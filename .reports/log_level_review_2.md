# Code Review: `fix/fatal_log_level_for_log_callback`

## Executive Summary

**Commit**: b3309cd9 — "log: pass fatal errors to callback if set"
**File changed**: `include/LightGBM/utils/log.h` (+9 / -3)

**What this PR does**: Routes `Fatal`-level log messages through the registered log callback (if set), instead of always writing directly to `stderr`. Previously, only `Debug/Info/Warning` honored the callback; `Fatal` always bypassed it.

**Verdict**: **APPROVE WITH RESERVATIONS**

The patch is **technically sound** from a C++ memory safety and thread safety perspective. The callback integration is **correct** from a Python bindings perspective. However, the devils-advocate review has identified **three actionable design concerns** that merit discussion:

1. **Behavioral regression**: Fatal errors are now written to stderr **OR** callback, not both. This loses a safety net for production systems.
2. **Silent message drops in Python**: Fatal messages are logged at INFO level in Python loggers, so they can be silently dropped by level-filtered loggers.
3. **Design debt**: The callback interface lacks severity metadata, forcing fragmented delivery and level-based routing in Python.

The patch is a minimal fix that extends an existing pattern to the Fatal path. The technical implementation is solid, but the design pattern it extends has limitations that are amplified for the more critical Fatal path.

---

## C++ Architect Review

### 1. Thread Safety of `GetLogCallBack()` When Called from Multiple Threads

**Verdict: SAFE — no issue.**

The callback pointer is stored in a `static THREAD_LOCAL` variable (`log.h:178-180`):

```cpp
static Callback &GetLogCallBack() {
    static THREAD_LOCAL Callback callback = nullptr;
    return callback;
}
```

Because the storage is `thread_local`, each thread has its own independent copy of the callback pointer. There is no shared mutable state between threads, so no data race is possible. The check-then-use pattern in the new code:

```cpp
if (GetLogCallBack() == nullptr) {
    // stderr path
} else {
    GetLogCallBack()("[LightGBM] [Fatal] ");
    // ...
}
```

...is safe. No other thread can modify this thread's callback pointer between the `nullptr` check and the invocations. There is no TOCTOU vulnerability.

**Important architectural note**: The `THREAD_LOCAL` design means `LGBM_RegisterLogCallback` (`c_api.cpp:975-978`) only sets the callback for the calling thread. Worker threads spawned by OpenMP or other C++ parallelism will have `callback == nullptr` and will write Fatal messages to stderr. This is a pre-existing design characteristic, not introduced by this change, and is consistent with how all other log levels already work.

### 2. Callback Invocation Pattern: 3 Separate Calls vs Single Formatted Message

**Verdict: Consistent with existing pattern — acceptable.**

The new Fatal code makes three separate callback invocations:

```cpp
GetLogCallBack()("[LightGBM] [Fatal] ");  // prefix
GetLogCallBack()(str_buf);                 // message body
GetLogCallBack()("\n");                    // newline terminator
```

This is identical to the pattern already used in `Write()` (`log.h:153-160`):

```cpp
snprintf(buf, kBufSize, "[LightGBM] [%s] ", level_str);
GetLogCallBack()(buf);          // prefix
vsnprintf(buf, kBufSize, format, val);
GetLogCallBack()(buf);          // message body
GetLogCallBack()("\n");         // newline terminator
```

The Python side handles this correctly via `_normalize_native_string` (`basic.py:265-279`), which accumulates chunks until it sees whitespace-only input (the `"\n"`), then flushes the assembled message. This works correctly for both the `Write()` and the new `Fatal()` paths.

**One subtle difference**: In `Fatal()`, the prefix is passed as a string literal `"[LightGBM] [Fatal] "`, while in `Write()` it's `snprintf`'d into `buf`. This is actually slightly better — it avoids the `snprintf` overhead and uses static string data. No functional difference.

### 3. Buffer Size Inconsistency: Fatal (1024) vs Write (512)

**Verdict: Not a problem — pre-existing and defensible.**

- `Fatal()` at `log.h:119`: `const size_t kBufSize = 1024;`
- `Write()` at `log.h:154`: `const size_t kBufSize = 512;`

In `Fatal()`, `str_buf[1024]` holds only the user's formatted message (the prefix is a string literal). In `Write()`, `buf[512]` is reused: first for the prefix, then overwritten with the message body via `vsnprintf`. The `Fatal` path benefits from a larger buffer because fatal messages often include file paths, line numbers, and diagnostic detail.

Both buffers are stack-allocated with a reasonable size. `vsnprintf`/`snprintf` are null-terminating and truncation-safe — they will never write beyond `kBufSize`. No memory safety issue.

**The new code does NOT use the `str_buf` for prefix formatting** (it passes a string literal directly), so the full 1024 bytes remain available for the message body — actually more efficient than `Write()`'s buffer reuse pattern.

### 4. Message Fragmentation and Atomicity in Multi-Threaded Contexts

**Verdict: Thread-safe for the callback path; minor pre-existing interleaving risk for stderr path.**

**Callback path** (new code): Because the callback pointer is `THREAD_LOCAL`, and each thread calls its own callback, there is no cross-thread fragmentation of callback messages. If two threads both call `Fatal()` simultaneously, each thread's 3-callback sequence executes independently on its own callback instance.

**stderr path** (unchanged code): `fprintf(stderr, "[LightGBM] [Fatal] %s\n", str_buf)` is a single `fprintf` call, which POSIX guarantees is atomic for output. So the stderr path is also atomic. This is unchanged by this PR.

**Cross-path interleaving**: If thread A has a callback and thread B does not, thread A's callback messages and thread B's stderr messages cannot interleave each other (they go to different destinations). No issue.

### 5. Exception Ordering After Callback Invocation

**Verdict: Correct — exception always thrown after callback completes.**

The execution flow at `log.h:131-138`:

```cpp
if (GetLogCallBack() == nullptr) {
    fprintf(stderr, ...);
    fflush(stderr);
} else {
    GetLogCallBack()("[LightGBM] [Fatal] ");
    GetLogCallBack()(str_buf);
    GetLogCallBack()("\n");
}
// ...
throw std::runtime_error(std::string(str_buf));  // line 138
```

The `throw` on line 138 executes after all three callback invocations complete. This means:

1. The callback receives the full message before the exception is thrown.
2. `str_buf` is still alive on the stack when `std::runtime_error` copies it.
3. The Python side's `_normalize_native_string` flushes the accumulated message on the third call (the `"\n"`), so the Python logger receives the complete message before the exception propagates.

**What if the callback itself throws?** In theory, if a C++ callback throws, the first callback invocation would unwind past the remaining two callbacks and past the `throw std::runtime_error` on line 138. The original runtime_error would never be thrown. However:

- The only consumer today is Python via ctypes `CFUNCTYPE`, which catches all Python exceptions, prints a traceback to stderr, and returns normally to C++. The `std::runtime_error` is always thrown.
- If a future C++ consumer registers a throwing callback, that would be a bug in the consumer, not in LightGBM.

No action needed for this PR.

### 6. Memory Safety with `vsnprintf`

**Verdict: SAFE — no issue.**

At `log.h:119-126`:

```cpp
const size_t kBufSize = 1024;
char str_buf[kBufSize];
va_start(val, format);
#ifdef _MSC_VER
    vsnprintf_s(str_buf, kBufSize, format, val);
#else
    vsnprintf(str_buf, kBufSize, format, val);
#endif
va_end(val);
```

- `vsnprintf(str_buf, kBufSize, ...)` writes at most `kBufSize - 1` characters plus a null terminator. No buffer overflow possible.
- `vsnprintf_s` (MSVC) has the same guarantee.
- `va_start`/`va_end` bracket is correct — `va_end` is called before the buffer is used for callbacks and for the exception.
- `str_buf` is on the stack and remains alive throughout the entire function body (callback calls and `throw`).
- `std::runtime_error(std::string(str_buf))` copies the buffer content into a heap-allocated string, so the exception object is safe even after `str_buf` goes out of scope.

This code is unchanged by the PR. No new memory safety concerns.

### 7. Header Guard Naming

**Verdict: Consistent with project convention — no issue.**

The header guard at `log.h:6-7`:

```cpp
#ifndef LIGHTGBM_INCLUDE_LIGHTGBM_UTILS_LOG_H_
#define LIGHTGBM_INCLUDE_LIGHTGBM_UTILS_LOG_H_
```

The guard includes `INCLUDE_` which reflects the file path `include/LightGBM/utils/log.h`. This is a reasonable naming convention — it encodes the full path relative to the repository root, avoiding collisions. This is consistent throughout the project.

### C++ Architect Summary

| Aspect | Verdict | Severity |
|--------|---------|----------|
| Thread safety | SAFE | No issue |
| Callback pattern consistency | OK | No issue |
| Buffer sizing | OK | No issue |
| Message atomicity | SAFE | No issue |
| Exception ordering | CORRECT | No issue |
| Memory safety | SAFE | No issue |
| Code style | CONSISTENT | No issue |

**Overall C++ verdict**: **APPROVE** — The change is a 9-line consistency fix that makes `Fatal()` follow the identical callback-or-stderr pattern already established in `Write()`. No memory safety, thread safety, or correctness issues found.

---

## Python Architect Review

### 1. API Contract Change for Python Callback Consumers

**Current behavior (master):** `Log::Fatal()` always writes to `stderr` regardless of whether a callback is registered. Python users who registered a custom logger via `register_logger()` (`basic.py:240-262`) would never see Fatal messages through their logger -- Fatal errors would silently go to stderr and then surface as a `LightGBMError` (raised by `_safe_call` at `basic.py:312-321` after the C API returns `-1`).

**New behavior (branch):** Fatal messages are now routed through the callback when one is registered, matching the behavior of Info/Warning/Debug messages. This is a **positive API contract change** -- it makes Fatal consistent with all other log levels.

**Impact on `register_logger()` consumers:** Users who call `register_logger()` with a custom logger will now see `[LightGBM] [Fatal] ...` messages arrive via their logger's `info` method (since `_log_native` at `basic.py:291-292` uses `_INFO_METHOD_NAME`). This is a minor semantic concern: Fatal-level messages will be logged at INFO level in the Python logger. However, this is **not a regression** -- before this change, Fatal messages were silently written to stderr and never reached the Python logger at all.

### 2. Fragmented Delivery (3 Calls) Handling

The change in `Log::Fatal()` uses the same 3-call pattern as `Write()`:
```cpp
// Fatal (new, line ~133-135 on branch):
GetLogCallBack()("[LightGBM] [Fatal] ");
GetLogCallBack()(str_buf);
GetLogCallBack()("\n");

// Write() (existing, line ~156-160):
GetLogCallBack()(buf);   // "[LightGBM] [Info] "
GetLogCallBack()(buf);   // formatted message
GetLogCallBack()("\n");
```

The Python side handles this via `_normalize_native_string` (`basic.py:265-279`), which buffers non-empty chunks and flushes them as a single concatenated string when it receives a whitespace-only message (the `"\n"`). The Fatal path uses an identical 3-call structure, so **`_normalize_native_string` will correctly reassemble Fatal messages** -- no Python-side changes needed.

**One subtle note:** The `_normalize_native_string` decorator checks `msg.strip() == ""`. The third callback call passes `"\n"`, which `strip()` reduces to `""`, triggering the flush. This works correctly.

### 3. Exception vs Callback Interaction in ctypes-based Calls

This is the **most critical aspect** of the review. The execution flow for a Fatal error is:

1. C++ code calls `Log::Fatal()`
2. **NEW:** Callback fires 3 times (prefix, message, newline) -- these invoke the Python `_log_callback` through ctypes
3. `throw std::runtime_error(str_buf)` executes (line 138, unchanged)
4. The exception propagates up the C++ call stack
5. `API_END()` macro catches it: `catch(std::exception& ex) { return LGBM_APIHandleException(ex); }` (`c_api.cpp:52`)
6. `LGBM_APIHandleException` calls `LGBM_SetLastError(ex.what())` and returns `-1`
7. Python's `_safe_call()` checks `ret != 0`, reads `LGBM_GetLastError()`, and raises `LightGBMError`

**Key safety observation:** The callback invocations (step 2) happen **before** the exception is thrown (step 3). This means the Python callback runs in a stable state -- there's no risk of the callback being invoked during exception unwinding. The ctypes callback is a normal synchronous C function call from Python's perspective.

**Potential concern -- callback exceptions:** If the Python callback itself raises an exception (e.g., a logger that fails), this would propagate through the ctypes boundary as an unhandled exception. However, this is an **existing risk** that applies to Info/Warning/Debug callbacks too -- it's not introduced by this change.

### 4. Test Coverage Gaps

**`test_callback.py`** (`tests/python_package_test/test_callback.py`): This file tests only the training callback infrastructure (early_stopping, log_evaluation, record_evaluation, reset_parameter). It does **not** test the native log callback mechanism at all.

**There are no tests for:**
- `_log_callback` / `_log_native` / `_normalize_native_string` behavior
- `register_logger()` with custom loggers receiving native messages
- Fatal-level messages being routed through the callback (the new behavior)
- The fragmented-delivery reassembly logic

**Recommendation:** The branch should add at least one test that:
1. Triggers a `Log::Fatal()` from the C++ side (e.g., by passing invalid parameters that cause a Fatal error)
2. Captures the logged output via a mock logger registered with `register_logger()`
3. Verifies the Fatal message appears in the mock logger's captured output
4. Verifies a `LightGBMError` is still raised

This would validate both the callback routing and the exception flow work correctly together.

### 5. Dask/Distributed Implications

**Finding:** The Dask module (`python-package/lightgbm/dask.py`) does **not** register its own log callback. It imports `_log_info` and `_log_warning` from `basic.py` (line 22) but these are Python-side helper functions -- they don't interact with the C++ callback mechanism.

The C++ `GetLogCallBack()` is `THREAD_LOCAL` (`log.h:179`). `LGBM_RegisterLogCallback` is called once at module import time on the main thread (`basic.py:304-306`). **This means worker threads spawned by C++ (e.g., OpenMP threads) will have `callback == nullptr`** and will use the stderr path for Fatal messages. This is the same behavior as for Info/Warning/Debug and is **consistent** -- it's not a regression.

For Dask distributed workers: each worker process imports `lightgbm` independently, which triggers the module-level callback registration. So distributed workers are covered.

### 6. R Binding Symmetry

The R binding is **unchanged** by this PR. In the R build path (`#ifdef LGB_R_BUILD`), `Log::Fatal()` always uses `REprintf` -- there is no callback mechanism for R. The R package does not call `LGBM_RegisterLogCallback` at all (confirmed by grep of `R-package/`).

This asymmetry is **pre-existing and intentional** -- R uses its own `Rprintf`/`REprintf` infrastructure. The change correctly limits itself to the `#ifndef LGB_R_BUILD` block.

### Python Architect Summary

| Aspect | Status | Notes |
|--------|--------|-------|
| API contract change | **OK** | Positive change; Fatal now consistent with other levels |
| Fragmented delivery | **OK** | Identical 3-call pattern; `_normalize_native_string` handles it |
| Exception vs callback | **OK** | Callback fires before throw; clean execution order |
| Test coverage | **GAP** | No tests for Fatal callback routing exist |
| Dask implications | **OK** | No regression; thread_local is consistent |
| R symmetry | **OK** | Unchanged; asymmetry is pre-existing and intentional |

**Overall Python verdict**: **APPROVE WITH TEST COVERAGE RECOMMENDATION** — The change is correct and safe from a Python bindings perspective. The only actionable finding is the test coverage gap.

---

## Devil's Advocate Cross-Review

### 1. Callback Exception Handling: A Real Danger Zone

The most serious unexamined risk in this change: **what happens if the callback throws during Fatal handling?**

The sequence in the patched code is:
```cpp
GetLogCallBack()("[LightGBM] [Fatal] ");  // call 1
GetLogCallBack()(str_buf);                 // call 2
GetLogCallBack()("\n");                    // call 3
throw std::runtime_error(std::string(str_buf));  // line 138
```

In the Python binding, the callback is a ctypes `CFUNCTYPE` wrapper around `_log_callback`. If the Python callback throws an exception (e.g., the Python logger raises, or there's a UnicodeDecodeError on malformed bytes), ctypes will print a traceback to stderr but **swallow the Python exception** and return normally to C++. So in practice, a Python callback exception won't prevent the `std::runtime_error` from being thrown. However:

- **If a non-Python C/C++ callback throws**: An exception thrown from call 1 or call 2 would prevent the `std::runtime_error` at line 138 from ever being reached. This means `Fatal()` would throw the *callback's* exception instead of the expected `std::runtime_error`. The `API_END()` macro catches `std::exception`, so it would still be caught, but with the **wrong error message** -- the callback's error rather than the actual fatal message. This is a semantic correctness bug for non-Python callers.

- **The existing `Write()` method has the same vulnerability** (lines 157-160), so this is not a regression per se, but it's worth noting that extending an already-fragile pattern to the more critical `Fatal` path amplifies the blast radius.

### 2. Fragmented Delivery (3 Calls) vs. Single Formatted Message

The C++ review noted that the `Write()` method already uses 3 separate calls. The patch mirrors this pattern for consistency. However, this consistency argument obscures a deeper design problem:

**The `_normalize_native_string` decorator in Python is the only thing making this work.** It buffers non-empty strings and flushes when it receives a whitespace-only string (the `"\n"`). This means:

- The callback contract is **implicitly** "messages come in fragments, terminated by a whitespace-only string." This is not documented anywhere in `c_api.h`. Any C/C++ consumer implementing a callback must reverse-engineer this protocol.

- **The 3-call pattern is more fragile for Fatal than for Write.** In `Write()`, if the process crashes between calls 1 and 3, you just lose a log line. In `Fatal()`, if something goes wrong between calls 1 and 3 (e.g., the callback mutates state, a signal arrives), you get a partial Fatal message in the buffer that never flushes, followed by a `std::runtime_error` throw. The Python `_normalize_native_string` would hold `["[LightGBM] [Fatal] ", str_buf]` in `msg_normalized` forever (well, until the exception unwinds and the message is lost).

- **Alternative**: The `Fatal` method already has the formatted `str_buf`. It could trivially build a single formatted string (`snprintf(buf, kBufSize, "[LightGBM] [Fatal] %s\n", str_buf)`) and make a single callback call. This would be safer, simpler, and wouldn't rely on the fragmentation protocol. The consistency argument is weaker than the reliability argument for the one code path that precedes a `throw`.

### 3. Behavioral Regression: stderr Is Now Skipped When Callback Is Set

This is the change most likely to cause real-world breakage, and the reviews should have flagged it prominently:

**Before this patch**: Fatal errors ALWAYS wrote to stderr (or R's error stream), regardless of whether a callback was registered. This was a safety net -- even if the callback was buggy, misconfigured, or logging to a file that got rotated, Fatal errors would appear on stderr.

**After this patch**: Fatal errors go to the callback OR stderr, never both.

This means:
- A user who registered a callback to customize Info/Warning formatting will **silently stop getting Fatal errors on stderr**. They may not even know this, because Fatal errors are rare and the callback was registered for a different purpose.
- In production systems, stderr is often captured by process managers (systemd, Docker, supervisord). Losing Fatal messages from stderr because someone registered a Python logging callback is a surprising behavioral change.
- The Python `_log_native` function logs everything at **INFO level** (`_INFO_METHOD_NAME`). So a Fatal error from C++ gets logged as `logger.info("[LightGBM] [Fatal] some error")`. If the Python logger has level set to WARNING or higher, **the Fatal message is silently dropped**. This is a genuine silent message loss scenario.

**Counterargument**: The `Write()` method already has this either/or behavior for Info/Warning/Debug. So the patch is "consistent." But Fatal is special -- it precedes an exception throw and is the last diagnostic output before error propagation. The safety argument for always writing Fatal to stderr is stronger than for other levels.

### 4. Risk of Silent Message Drops

Building on the behavioral regression above, here is a concrete scenario:

```python
import logging
import lightgbm as lgb

# User configures Python logging with WARNING threshold
logging.basicConfig(level=logging.WARNING)
logger = logging.getLogger("lightgbm")
logger.setLevel(logging.WARNING)
lgb.register_logger(logger)

# Now Fatal messages from C++ will be:
# 1. Sent to callback (3 fragments)
# 2. Reassembled by _normalize_native_string
# 3. Passed to _log_native which calls logger.info()
# 4. DROPPED by Python logging because INFO < WARNING threshold
# 5. stderr output is SKIPPED because callback is set
# 6. std::runtime_error is thrown, caught by API_END(), stored in LastError
# 7. Python raises LightGBMError from _safe_call
#
# The error IS propagated via the exception, but the diagnostic log line is LOST.
```

The exception itself still propagates (line 138 still throws), so the error is not truly "silent" -- `_safe_call` will raise `LightGBMError`. But the **log message** -- which may contain additional context beyond the exception message -- is lost. More importantly, in scenarios where the exception is caught and swallowed (defensive code), the log line was the only diagnostic.

**However**: One could argue that the exception message (`str_buf`) contains the same content as the log line, so the log line is redundant. This is mostly true, but the `[LightGBM] [Fatal]` prefix and the stderr destination are valuable for post-mortem analysis (grepping logs, filtering by severity).

### 5. Design Question: Minimal Fix or Larger Interface Issue?

This patch exposes a fundamental design tension in the callback interface:

**The callback has no level parameter.** The signature is `void (*)(const char*)`. The callback receives raw string fragments with no structured metadata about severity. The only way to determine severity is to parse the `[LightGBM] [Fatal]` prefix from the string content. This means:

- The Python side cannot route Fatal messages to `logger.error()` or `logger.critical()` -- everything goes through `_INFO_METHOD_NAME` (point 3 above).
- Third-party callback implementations cannot implement level-based filtering or routing.
- The fragmented delivery makes even prefix-parsing unreliable (you'd have to buffer and inspect the first fragment).

**This is the real issue**: The patch is a minimal fix for "Fatal doesn't use callback," but the callback interface itself is too primitive to properly handle severity levels. A proper fix would either:
(a) Add a level-aware callback: `void (*)(int level, const char* msg)` -- but this is a C API break.
(b) Always write Fatal to stderr AND send to callback -- preserving the safety net.
(c) At minimum, format Fatal as a single string before calling the callback.

The patch chooses none of these and instead applies the existing (flawed) pattern to a more critical code path.

### Devil's Advocate Summary

| Risk | Severity | Likelihood | Suggested Mitigation |
|------|----------|------------|----------------------|
| Callback exception replaces fatal message | **Medium** | Low | Document ctypes exception handling; document callback contract |
| Fragmented delivery is fragile for Fatal | **Medium** | Low | Consider single-call formatting for Fatal |
| stderr output lost (behavioral regression) | **HIGH** | High | Write Fatal to both stderr AND callback |
| Silent message drops in Python loggers | **HIGH** | Medium | Upgrade callback contract with severity, or route Fatal to `logger.error()` |
| Callback interface design debt | **Medium** | N/A | Consider API evolution in future |

**Overall devil's advocate verdict**: **CONDITIONAL APPROVAL** — The patch is technically correct and consistent with existing patterns, but it extends a fragile pattern to a more critical code path (Fatal precedes an exception throw). The recommended mitigations are:

1. **Preserve stderr output**: Write Fatal to stderr even when callback is set. This is the lowest-risk change and preserves the safety net.
2. **If** stderr must be skipped, at minimum format Fatal as a single callback invocation rather than 3 fragments.
3. **Python-side improvement**: Route Fatal messages to `logger.error()` in `_log_native()` instead of `logger.info()`, or enhance the callback contract with severity metadata.

---

## Consolidated Recommendations (Ranked by Severity)

### 🔴 CRITICAL CONCERNS (Recommend Resolution Before Merge)

**1. Silent Fatal Message Drops in Python Loggers**

**Severity**: HIGH
**Likelihood**: Medium (depends on user's logging config)
**Impact**: A Fatal error from C++ can be silently dropped by Python logging if the Python logger's level is set above INFO.

**Current scenario:**
- User registers a Python logger with `logging.WARNING` threshold
- C++ Fatal message routed to callback
- `_log_native()` calls `logger.info()` (not `logger.error()`)
- Message silently dropped because INFO < WARNING
- Exception is still raised, but diagnostic output is lost

**Recommendation:**
- Modify `_log_native()` to route `[LightGBM] [Fatal]` prefix to `logger.error()` or `logger.critical()` instead of `logger.info()`
- Document this behavior in docstrings

---

**2. Behavioral Regression: stderr Output Skipped When Callback Set**

**Severity**: HIGH
**Likelihood**: High (affects all callback users)
**Impact**: Fatal errors no longer guaranteed to reach stderr when callback is registered. Production systems relying on stderr capture (systemd, Docker, supervisord) may miss Fatal diagnostics.

**Current scenario:**
- User registers callback for Info/Warning customization
- Fatal error occurs
- Message goes to callback, never to stderr
- stderr capture mechanism (process manager, log aggregator) never sees the message

**Recommendation:**
- **Preferred**: Write Fatal to stderr **AND** callback (always)
- **Alternative**: Document this breaking change prominently in release notes
- **Fallback**: At minimum, consider this for next major version

---

### 🟡 MODERATE CONCERNS (Should Address)

**3. Test Coverage Gap for Fatal Callback Routing**

**Severity**: MEDIUM
**Likelihood**: High (issue exists now)
**Impact**: New behavior (Fatal → callback) is untested. Regressions in callback reassembly or exception flow could slip through.

**Recommendation:**
- Add test in `test_callback.py` that:
  1. Triggers a `Log::Fatal()` from C++ (e.g., invalid parameter)
  2. Verifies callback received the message via mock logger
  3. Verifies exception was still raised

---

**4. Fragmented Delivery Is More Fragile for Fatal**

**Severity**: MEDIUM
**Likelihood**: Low
**Impact**: If something interrupts between the 3 callback calls (signal, callback state mutation), a partial Fatal message could get stuck in `_normalize_native_string` buffer.

**Recommendation:**
- **Preferred**: Format Fatal as single string before callback:
  ```cpp
  snprintf(formatted, kBufSize, "[LightGBM] [Fatal] %s\n", str_buf);
  GetLogCallBack()(formatted);
  ```
- **Acceptable if changed to stderr + callback**: Fragmentation is less critical if message always reaches stderr.

---

### 🟢 MINOR CONCERNS (Low Priority)

**5. Callback Exception Handling Documentation**

**Severity**: MEDIUM
**Likelihood**: Low (only affects non-Python C/C++ callback users)
**Impact**: If a C/C++ callback throws, the Fatal exception message is replaced. This is confusing for callback implementers.

**Recommendation:**
- Document callback contract in `c_api.h`: "Callbacks must not throw exceptions"
- Consider wrapping callback invocations in try-catch for safety

---

## Summary Table

| Recommendation | Priority | Effort | Risk of Merge Without It |
|---|---|---|---|
| Route Fatal to `logger.error()` in Python | HIGH | Low | Silent message drops in production |
| Write Fatal to stderr + callback | HIGH | Low | Loss of critical diagnostic output |
| Add Fatal callback test | MEDIUM | Low | Untested new behavior |
| Use single-call formatting for Fatal | MEDIUM | Medium | Fragmentation issues with interrupts |
| Document callback contract | MEDIUM | Low | Confusion for callback implementers |

---

## Final Verdict

**APPROVE WITH CRITICAL CONCERNS**

The code is **technically sound** and **passes all safety checks**. However, the design choices introduce two high-severity behavioral changes that merit discussion:

1. **Silent message drops** in Python loggers
2. **stderr output loss** compared to current behavior

These concerns are not blockers but should be resolved before or immediately after merge:
- **Highest priority**: Route Fatal to `logger.error()` in Python (simple, high impact)
- **High priority**: Decide on stderr behavior (double-write or accept breaking change)
- **Medium priority**: Add test coverage for Fatal callback path

The patch is a reasonable consistency fix, but Fatal is a special case that deserves slightly higher safety standards than the Write() pattern it's based on.

---

**Generated by three-agent code review: cpp-architect, python-architect, devils-advocate**
**Review date**: 2026-02-20
