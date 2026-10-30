# jadx: `continue` to an outer loop across two loop levels is decompiled as a plain `break`

## Branch

- Fork: `b84330808/jadx`
- Branch: `claude/upbeat-rubin-a755cz`
- Commit: `29619192efa46e65d2b037e39f03808a9f93edb9`
- Base: `upstream/master` at `15cba22e6` (`feat(gui): scan opened jadx plugins for dangerous API usage (PR #3036)`)
- Author: `b84330808 <b84330808@gmail.com>`, trailer `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`
- Files: `jadx-core/src/main/java/jadx/core/dex/visitors/regions/PostProcessRegions.java` (+43),
  `jadx-core/src/test/java/jadx/tests/integration/loops/TestBreakWithLabel2.java` (new, 77 lines)

## PR title

```
fix(core): label 'break' to an outer loop placed inside a nested loop
```

## PR body (repo template: `.github/pull_request_template.md`)

````markdown
:exclamation: Please review the [guidelines for contributing](https://github.com/skylot/jadx/blob/master/CONTRIBUTING.md#Pull-Request-Process)

### Description

A `continue` to an outer loop from a loop nested two levels deep is decompiled as an unlabeled `break`, which changes what the code does:

```java
int c = 0;
outer: for (int i = 0; i < n; i++) {
    for (int j = 0; j < n; j++) {
        for (int k = 0; k < n; k++) {
            if (i + j + k == 3) {
                continue outer;
            }
            c++;
        }
    }
}
return c;
```

is decompiled as:

```java
for (int i = 0; i < n; i++) {
    for (int j = 0; j < n; j++) {
        for (int k = 0; k < n; k++) {
            if (i + j + k == 3) {
                break;      // only leaves the 'k' loop
            }
            c++;
        }
    }
}
```

For `n = 3` the original returns 8 and the decompiled code returns 13. The `while` form (jump back to the outer loop condition) is affected the same way.

`LoopRegionMaker` handles the jump correctly as an exit from the middle (`j`) loop and adds a `break` for that loop. But the `break` sits inside the innermost (`k`) loop, and `addBreakLabel()` only adds a label when the exit target is outside all loops. Here the target is in the outer loop, so the `break` has no label and leaves only the `k` loop.

The fix adds a small pass in `PostProcessRegions` that runs after regions are built (only for methods with more than one loop). It walks the region tree, and when a loop `break` is enclosed by a loop region nested inside the loop it exits, it adds a `LoopLabelAttr` to the `break` and to that loop's start, the same way `addBreakLabel()` does. Output is now:

```java
for (int i = 0; i < n; i++) {
    loop1: for (int j = 0; j < n; j++) {
        for (int k = 0; k < n; k++) {
            if (i + j + k == 3) {
                break loop1;
            }
            c++;
        }
    }
}
```

`break`s that already have a label, and `break`s that are not inside a nested loop, are not changed.

Added `TestBreakWithLabel2` (`for` and `while` variants, `check()` runs the decompiled code), with one test for the default input and one with `useJavaInput()`.

Found by differential testing (compile, decompile, recompile, run and compare output).

I used Claude Code to write this change.
````

## Fail-before / pass-after

Command (see the "Environment limits" section for why the init script and `TEST_INPUT_PLUGIN=java` were needed):

```
TEST_INPUT_PLUGIN=java ./gradlew --no-scan -I no-google.gradle :jadx-core:test --tests 'jadx.tests.integration.loops.TestBreakWithLabel2'
```

### Fail-before (fix reverted with `git stash`, new test kept)

```
TestBreakWithLabel2 > testJava() FAILED
    org.opentest4j.AssertionFailedError:
    expected: 8
     but was: 13
        at jadx.tests.integration.loops.TestBreakWithLabel2$TestCls.check(TestBreakWithLabel2$TestCls.java:45)
        at java.base@21.0.12.1/java.lang.reflect.Method.invoke(Method.java:580)
        at app//jadx.tests.api.compiler.TestCompiler.invoke(TestCompiler.java:123)
        at app//jadx.tests.api.IntegrationTest.invoke(IntegrationTest.java:554)
        at app//jadx.tests.api.IntegrationTest.lambda$runDecompiledAutoCheck$4(IntegrationTest.java:481)

TestBreakWithLabel2 > test() FAILED
    org.opentest4j.AssertionFailedError:
    expected: 8
     but was: 13
        at jadx.tests.integration.loops.TestBreakWithLabel2$TestCls.check(TestBreakWithLabel2$TestCls.java:45)
        ...
        at app//jadx.tests.api.IntegrationTest.lambda$runDecompiledAutoCheck$4(IntegrationTest.java:481)

2 tests completed, 2 failed
> Task :jadx-core:test FAILED
```

(`check()` fails on the recompiled decompiled class: `testFor(3)` returns 13 instead of 8.)

### Pass-after (with fix)

```
TestBreakWithLabel2: tests="2" skipped="0" failures="0" errors="0"  (test(), testJava())
BUILD SUCCESSFUL in 5s
```

Decompiled code printed by the test (`-i`):

```java
public int testFor(int n) {
    int c = 0;
    for (int i = 0; i < n; i++) {
        loop1: for (int j = 0; j < n; j++) {
            for (int k = 0; k < n; k++) {
                if (i + j + k == 3) {
                    break loop1;
                }
                c++;
            }
        }
    }
    return c;
}

public int testWhile(int n) {
    int c = 0;
    int i = 0;
    while (i < n) {
        i++;
        int j = 0;
        loop1: while (j < n) {
            j++;
            int k = 0;
            while (k < n) {
                k++;
                if (i + j + k == 4) {
                    break loop1;
                }
                c++;
            }
        }
    }
    return c;
}
```

### Full `:jadx-core:test` (java input), base vs fix

| | tests | failed | skipped |
|---|---|---|---|
| base (fix reverted, new test kept) | 1039 | 21 | 281 |
| fix | 1039 | 19 | 281 |

The only difference in the failure sets is the 2 new tests (`TestBreakWithLabel2#test`, `#testJava`): they fail on base and pass with the fix. The 19 remaining failures are the same on base and fix, are unrelated to this change, and come from running the suite with java input:
`TestArith3`, `TestConditions14`, `TestInline2`, `TestBreakInComplexIf` (x2), `TestCaseSensitiveChecksTurkishLocale`, `TestSwitch4`, `TestSwitchBreak`, `TestSwitchWithFallThroughCase`, `TestSwitchWithFallThroughCase2`, `TestFinally`, `TestFinally2`, `TestFinallyExtract` (x2), `TestInlineInCatch`, `TestTryAfterDeclaration#test62`, `TestTryCatch7#testNoDebug`, `TestTryCatchFinally6` (x2).

### Style

`./gradlew --no-scan :jadx-core:spotlessCheck --rerun-tasks :jadx-core:checkstyleMain :jadx-core:checkstyleTest`: BUILD SUCCESSFUL.
The code uses only Java 11 API (no pattern matching), as CONTRIBUTING.md requires.

### Differential testing (javac -> jadx java-input -> javac -> run -> compare)

- Hand-written repros (`for`, `while`, `do-while`, for-each, 4-level nesting, `continue a` and `continue b` in one loop, code after the inner loop): 8 programs wrong on base are correct with the fix. 3 more still fail on both, only because they also contain unrelated pre-existing bugs (findings 1 and 3 below).
- 120 generated programs (2 to 4 nested `for`/`while`/`do` loops with random labeled and unlabeled `break`/`continue`):
  - 87 correct on both
  - 16 wrong on base, correct with the fix
  - 15 wrong on both (10 "Method not decompiled", 5 other loop bugs, see below)
  - 2 don't recompile on both
  - **0 regressions** (no case that is correct on base and wrong with the fix).
- In all cases, the only differences in decompiled output between base and fix are added `loopN:` labels and `break` becoming `break loopN`.

## Duplicate search (all PR states)

- Every upstream PR ref from #2700 up (159 refs, any state, fetched with `git fetch upstream refs/pull/N/head`) plus #2383: diffed against the merge base, looking for changes to `PostProcessRegions`, `LoopRegionMaker`, `LoopLabelAttr`, `RegionGen`, `InsnGen`, or added lines with `LOOP_LABEL`, `LoopLabelAttr`, `addBreakLabel`, `getLoopLabel`, `break loopN`, or a labeled `continue`.
  - #2784 (merged, "feat: graph views, code pane sync, and more"): the hits come from master merged into it, not a fix.
  - #3041 (open, ours): `LoopRegionMaker`, a different case (break in a loop after a switch). It doesn't touch `PostProcessRegions` or labels.
  - #2905, #2982, #3003, #3042: touch `RegionGen`/`InsnGen` for unrelated things (try comments, varargs, field aliases), with no loop/label/break/continue lines.
- All 29 open PRs (listed from the GitHub PR page) are covered by the scan above.
- GitHub searches over issues and PRs, all states: `continue outer` (0), `labeled continue` (1: #22 "Break From Labeled Block", a labeled *block* with `if`, not loops), `label break nested` (0), `is:pr nested loop` (#2936 if-region targets, #1532 mapping-io), `is:pr break` (59, no labels or nested-loop fix), `continue loop` (22, incl. #43 "Premature Break" = endless loop with try/catch, different).
- Upstream history: the only related commit is `8a4ec47b9` (2014) "core: support break with label for simple cases", which added `addBreakLabel()` for exits outside all loops.

## Environment limits (please re-run locally before opening the PR)

- The cloud network policy denied `dl.google.com` (Google Maven). `smali`, `baksmali`, `r8` (d8) and `aapt2-proto` are only there, so `jadx-dex-input`, `jadx-smali-input` and `jadx-java-convert` could not build. That means **the dex input path (the default `test()` in CI) was not run here**.
- To run the core tests, a Gradle init script (not committed) removed those three `testImplementation` deps from `:jadx-core` and excluded `jadx/api/JadxDecompilerTest.java` (it imports `DexInputPlugin`), and tests ran with `TEST_INPUT_PLUGIN=java`. So `TestBreakWithLabel2#test()` ran with java input here too.
- Please run locally before opening: `./gradlew clean build dist` (CONTRIBUTING.md), or at least `./gradlew :jadx-core:test` with the default dex input. Check that `TestBreakWithLabel2` passes with d8 input (it asserts `countString(2, "break loop")` plus the runtime `check()`).
- GitHub API reads for `skylot/jadx` are not enabled in this session, so the duplicate search used `git` PR refs and the public GitHub web pages.

## Other findings (not fixed, all fail the same on base)

1. Post-increment `while (k++ < n)` innermost loop loses its exit and becomes `while (true)` with no `break` (infinite loop):
   ```java
   static int whileTrue(int n) {
       int c = 0, i = 0;
       outer: while (true) {
           i++;
           if (i > n) break;
           int j = 0;
           while (j++ < n) {
               int k = 0;
               while (k++ < n) {
                   if ((i ^ j ^ k) == 2) continue outer;
                   c += k;
               }
           }
       }
       return c;
   }
   ```
2. Two levels: `break` plus `continue L0` in the inner loop, with code after it, is turned into a wrong `do { ... break; } while (...)`:
   ```java
   L0: for (int v0 = 0; v0 < n; v0++) {
       c += 2;
       int v1 = 0;
       while (v1 < n) {
           v1++;
           c += 9;
           if ((v1 * 5 + c) % 7 == 0) break;
           if ((v1 * 3 + c) % 4 == 1) continue L0;
       }
       c ^= 5;
   }
   ```
3. A labeled `continue outer` inside a `try` in nested loops: StackOverflowError in `RegionMakerVisitor` ("Method not decompiled").
4. A switch in a loop with `continue` / `break outer` in cases: an extra `break;` after `continue;` and a doubled `break; break;` give an "unreachable statement" recompile error. This is near our #3041 area, so check for overlap first.
5. Nested try/catch/finally with `return` in the try: the decompiled method is missing a `return` (recompile error).
6. Java input: records decompile as `final class X extends Record` (doesn't recompile).
