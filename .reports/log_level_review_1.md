# Code Review Report: `fix/fatal_log_level_for_log_callback`

**Branch**: `fix/fatal_log_level_for_log_callback`
**Commit**: b3309cd9 — "log: pass fatal errors to callback if set"
**File changed**: `include/LightGBM/utils/log.h` (+9 / -3 lines)
**Date**: 2026-02-20
**Review panel**: C++ Architect, Python Architect, Devil's Advocate (cross-review)

---

## 1. Change Summary

The `Log::Fatal()` method was modified so that when a log callback is registered (via `LGBM_RegisterLogCallback`), fatal messages are routed through the callback instead of being written directly to `stderr`. After dispatching through the callback, the code still throws `std::runtime_error` — the exception flow is unchanged.

```diff
 // R code should write back to R's error stream,
-// otherwise to stderr
+// otherwise to stderr or callback if set
 #ifndef LGB_R_BUILD
-    fprintf(stderr, "[LightGBM] [Fatal] %s\n", str_buf);
-    fflush(stderr);
+    if (GetLogCallBack() == nullptr) {
+      fprintf(stderr, "[LightGBM] [Fatal] %s\n", str_buf);
+      fflush(stderr);
+    } else {
+      GetLogCallBack()("[LightGBM] [Fatal] ");
+      GetLogCallBack()(str_buf);
+      GetLogCallBack()("\n");
+    }
 #else
     REprintf("[LightGBM] [Fatal] %s\n", str_buf);
     R_FlushConsole();
```

**Intent**: Before this change, `Fatal` was the only log level that never went through the registered callback. Python users with custom loggers via `register_logger()` never saw Fatal messages in their logger — they only appeared on `stderr`. This change makes `Fatal` consistent with `Info`/`Warning`/`Debug`, which already route through the callback.

---

## 2. Consolidated Findings

### 2.1 Fatal messages logged at Python INFO level

| Attribute | Value |
|---|---|
| Severity | **Medium** |
| Introduced by this change? | Amplified (pre-existing for Warning) |
| Reviewers | C++ (Medium), Python (High), Devil's Advocate (Low-Medium) |

**Description**: The Python callback chain converges into `_log_native` (`python-package/lightgbm/basic.py:290-292`), which dispatches everything via `_LOGGER.info()`. Fatal-level messages become indistinguishable from informational messages in Python logging. Warning messages from C++ already suffer the same issue (`INFO | [LightGBM] [Warning] ...` is visible in the existing test expectations at `test_utilities.py:45`).

**Mitigating factor**: `Fatal()` still throws `std::runtime_error`, which is caught by `API_END()` and surfaces as `LightGBMError` in Python. The exception is the primary error reporting mechanism — the callback message is supplementary diagnostic output. No fatal error is silently swallowed.

**Risk**: Users who set Python logging level to `WARNING` or above will lose the Fatal diagnostic text from the callback, but will still get the exception with the error message.

**Recommendation**: Acceptable for this PR. Level-aware routing (parsing the `[LightGBM] [Level]` prefix in `_log_native`, or adding `error_method_name` to `register_logger()`) should be a follow-up PR that addresses the whole logging design holistically.

---

### 2.2 Callback exception behavior — reviewer disagreement resolved

| Attribute | Value |
|---|---|
| Severity | **Low** |
| Introduced by this change? | Yes |
| Reviewers | C++ (Medium — recommends try/catch), Devil's Advocate (Low — refutes) |

**Description**: The C++ architect raised concern that if the callback throws a C++ exception, the `std::runtime_error` would never be thrown. The devil's advocate **refuted** this for the Python binding: ctypes `CFUNCTYPE` catches all Python exceptions, prints a traceback to stderr, and returns normally. The `std::runtime_error` is always thrown regardless of Python callback failures.

**Where the concern IS valid**: If a non-Python consumer registers a raw C++ function as the callback that throws. No such consumer exists today.

**Recommendation**: No try/catch wrapper needed. If a C++ callback consumer appears in the future, this can be revisited.

---

### 2.3 Loss of stderr output for Fatal when callback is registered

| Attribute | Value |
|---|---|
| Severity | **Low** |
| Introduced by this change? | Yes |
| Reviewers | Python (Medium), Devil's Advocate (Low) |

**Description**: Before this change, `Fatal()` always wrote to stderr. After, with a callback registered, Fatal goes through the callback but NOT to stderr. The Python architect suggested belt-and-suspenders (both stderr and callback).

**Mitigating factors**:
- This is consistent with how `Write()` already works for all other log levels (either-or, not both).
- The exception still surfaces with the error message.
- Worker threads (where callback is `nullptr` due to `THREAD_LOCAL`) still write Fatal to stderr — accidental belt-and-suspenders for the multi-threaded case.

**Recommendation**: Consistency with the existing pattern is the right choice. If the project wants both-stderr-and-callback for Fatal, that is a design decision for the whole logging system.

---

### 2.4 `_normalize_native_string` accumulator state

| Attribute | Value |
|---|---|
| Severity | **Low** |
| Introduced by this change? | Pre-existing, newly relevant |
| Reviewers | Python (Medium), Devil's Advocate (Low) |

**Description**: The `_normalize_native_string` decorator (`basic.py:265-279`) buffers chunks in `msg_normalized` until it sees whitespace-only string (`"\n"` stripped = `""`). If callback #3 is never reached, stale fragments remain. However, ctypes swallows Python exceptions and returns normally, so all three callbacks complete. The message is correctly assembled as `"[LightGBM] [Fatal] <message>"`.

**Recommendation**: No action needed for this PR. The accumulator works correctly for the three-call pattern.

---

### 2.5 Thread-local callback invisible to worker threads

| Attribute | Value |
|---|---|
| Severity | **Low** |
| Introduced by this change? | Pre-existing |
| Reviewers | C++ (Medium), Python (Low), Devil's Advocate (Info) |

**Description**: `LGBM_RegisterLogCallback` is called on the Python main thread. Worker threads spawned by the C++ core (e.g., OpenMP) have `callback == nullptr` and write Fatal to stderr. This is consistent with how all other log levels work.

**Silver lining**: Worker-thread Fatal messages always go to stderr regardless of callback registration — accidental robustness.

**Recommendation**: Document as known limitation. Not a blocker.

---

### 2.6 TOCTOU on callback pointer is safe

| Attribute | Value |
|---|---|
| Severity | **Info (no issue)** |
| Introduced by this change? | N/A |
| Reviewers | C++ (Info — correct) |

The check-then-use pattern on `GetLogCallBack()` is safe because the callback is `THREAD_LOCAL` — no other thread can modify this thread's pointer.

---

### 2.7 Stack buffer lifetime is safe

| Attribute | Value |
|---|---|
| Severity | **Info (no issue)** |
| Introduced by this change? | N/A |
| Reviewers | C++ (Info — correct) |

`str_buf[1024]` is on the stack and alive during all callback invocations. The subsequent `throw std::runtime_error(std::string(str_buf))` copies into a heap-allocated string.

---

### 2.8 Code style is consistent

| Attribute | Value |
|---|---|
| Severity | **Info (no issue)** |
| Introduced by this change? | N/A |
| Reviewers | C++ (Info — correct) |

The new code mirrors the exact pattern in `Write()`. Indentation, brace style, comment update, and callback invocation pattern are all consistent.

---

### 2.9 UTF-8 decode in Python callback

| Attribute | Value |
|---|---|
| Severity | **Low** |
| Introduced by this change? | Pre-existing |
| Reviewers | Python (Low) |

`_log_callback` (`basic.py:295-297`) uses `msg.decode("utf-8")` which could raise on non-UTF-8 bytes. In practice, LightGBM error messages are ASCII with format substitutions. A user-supplied file path with non-UTF-8 encoding could theoretically trigger this.

**Recommendation**: Nice-to-have fix (`errors="replace"`) in a separate PR — applies to the pre-existing `_log_callback`, not this diff.

---

### 2.10 Missing test coverage

| Attribute | Value |
|---|---|
| Severity | **Medium** |
| Introduced by this change? | Yes (gap) |
| Reviewers | Python (Medium), Devil's Advocate (valuable but not blocking) |

No tests exist for Fatal messages going through the callback. Recommended test scenarios:

1. Trigger `Log::Fatal` via C API with invalid parameters, verify message appears in custom logger AND `LightGBMError` is raised
2. Verify `_normalize_native_string` correctly assembles the three-chunk Fatal message
3. Verify Fatal does not write to stderr when callback is registered
4. Verify subsequent non-fatal messages are not corrupted after Fatal
5. Verify accumulator state is clean after Fatal flow

**Mitigating factor**: The Python package always registers the callback at import time, so the existing test suite implicitly exercises the callback path for all non-Fatal levels. The Fatal path follows the identical three-call pattern.

**Recommendation**: Adding at least one integration test would be valuable but should not block merge.

---

### 2.11 Re-entrancy risk

| Attribute | Value |
|---|---|
| Severity | **Low** |
| Introduced by this change? | Pre-existing |
| Reviewers | C++ (Low), Python (Medium), Devil's Advocate (Low) |

If the callback triggers another `Log::Fatal` (e.g., user's custom logger re-enters LightGBM), infinite recursion until stack overflow. Requires pathological user behavior. Same risk already exists in `Write()`.

---

### 2.12 Buffer size discrepancy (1024 vs 512)

| Attribute | Value |
|---|---|
| Severity | **Info** |
| Introduced by this change? | Pre-existing |
| Reviewers | C++ (Low), Python (Low) |

`Fatal()` uses 1024-byte buffer, `Write()` uses 512. Not a problem — `Fatal` messages benefit from more space for diagnostic detail. The prefix is sent as a string literal, not consuming buffer space.

---

### 2.13 R build path correctly untouched

| Attribute | Value |
|---|---|
| Severity | **Info** |
| Introduced by this change? | N/A |
| Reviewers | Python (Info — correct) |

The `#ifdef LGB_R_BUILD` branch is unchanged. R builds continue using `REprintf`.

---

## 3. Devil's Advocate — Blind Spots Identified

Both specialist reviewers missed:

| Blind Spot | Assessment |
|---|---|
| Signal handling context for `Fatal()` | Non-issue. `fprintf` was already not async-signal-safe. LightGBM does not install signal handlers. |
| `std::terminate` during stack unwinding | Non-issue. Unchanged by this diff — the throw on line 138 is identical. |
| ODR violations from header-defined static locals | Non-issue. C++17 guarantees one instance per thread for implicit-inline class methods with `static thread_local`. |
| Blocking callbacks (network loggers) | Pre-existing concern for `Write()`. Not introduced by this diff. |
| Performance of 3 callbacks vs 1 fprintf | Non-issue. `Fatal()` fires at most once per program execution (it throws). You do not optimize the crash path. |
| ctypes swallows Python exceptions | **Key insight missed by C++ reviewer.** ctypes `CFUNCTYPE` catches Python exceptions, prints traceback, returns normally. The `std::runtime_error` is ALWAYS thrown regardless of callback behavior. This refutes the C++ reviewer's recommendation for a try/catch wrapper. |

---

## 4. Summary Verdicts

### Per Reviewer

| Reviewer | Verdict | Key Insight |
|---|---|---|
| **C++ Architect** | Approve with minor changes | Sound analysis; try/catch recommendation incorrect for Python/ctypes context |
| **Python Architect** | Approve with changes | Thorough; somewhat over-scoped — many findings are pre-existing design debt |
| **Devil's Advocate** | **Approve as-is** | The diff is a consistency fix; pre-existing design debt should be separate PRs |

### Risk Assessment

| Risk | Level |
|---|---|
| Data loss or silent failure | **None** — exception always thrown |
| Crash or undefined behavior | **None** — same callback mechanism as Write() |
| Confusing log output | **Low** — Fatal appears as INFO level, but followed by exception |
| Behavioral regression | **Minimal** — only affects users grepping stderr for `[Fatal]` while also having Python package imported |

### Consolidated Recommendation

| Action | Priority | Scope |
|---|---|---|
| **Merge this PR as-is** | Now | This PR |
| Add test for Fatal through callback | Nice-to-have | This PR or follow-up |
| Level-aware routing in `_log_native` | Follow-up PR | Holistic logging redesign |
| Add `error_method_name` to `register_logger()` | Follow-up PR | API enhancement |
| UTF-8 `errors="replace"` in `_log_callback` | Follow-up PR | Defensive coding |

---

## 5. Final Assessment

**The change is safe to merge.** It is a 9-line consistency fix that makes `Fatal()` follow the same callback-or-stderr pattern already used by `Write()` for Info/Warning/Debug. The exception flow is unchanged, no error is silently swallowed, code style is consistent, and the three-call callback pattern works correctly with Python's `_normalize_native_string`. The pre-existing design debt around level-aware log routing is real but orthogonal to this diff.
