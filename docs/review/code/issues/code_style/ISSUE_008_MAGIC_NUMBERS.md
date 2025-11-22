# Issue: Magic Numbers in Code

**Severity**: Low
**Category**: Code Style / Maintainability
**Files Affected**: Various files

---

## Description

The codebase contains several magic numbers that should be replaced with named constants for better readability and maintainability.

---

## Examples Found

### Example 1: Buffer Sizes

**File**: `/home/user/LightGBM/include/LightGBM/c_api.h`
**Line**: 1646

```cpp
// Current
static THREAD_LOCAL char err_msg[512] = "Everything is fine";

// Recommended
constexpr size_t kErrorBufferSize = 512;
static THREAD_LOCAL char err_msg[kErrorBufferSize] = "Everything is fine";
```

### Example 2: Log Buffer Size

**File**: `/home/user/LightGBM/include/LightGBM/utils/log.h`
**Lines**: 119, 154

```cpp
// Current
const size_t kBufSize = 1024;
char str_buf[kBufSize];

// This is already good - uses named constant
```

### Example 3: Alignment Values

**File**: `/home/user/LightGBM/include/LightGBM/meta.h`
**Line**: 72

```cpp
// Current - well-named constant
const size_t kAlignedSize = 32;  // Good!
```

---

## Guidelines

| Type | Acceptable | Better |
|------|------------|--------|
| `512` for buffer size | No | `kBufferSize = 512` |
| `32` for alignment | No | `kAlignedSize = 32` |
| `0`, `1`, `-1` | Yes | Commonly understood |
| Loop bounds from data | Yes | `for (int i = 0; i < n; ++i)` |
| Mathematical constants | No | `constexpr double kPi = 3.14159...` |

---

## Recommended Constants

```cpp
// In a constants header or config.h
namespace LightGBM {
namespace Constants {

// Buffer sizes
constexpr size_t kErrorBufferSize = 512;
constexpr size_t kLogBufferSize = 1024;
constexpr size_t kLineBufferSize = 4096;

// Memory alignment
constexpr size_t kCacheLineSize = 64;
constexpr size_t kSimdAlignment = 32;

// Default values (already in config.h as parameters)
// These serve as documentation for the defaults

}  // namespace Constants
}  // namespace LightGBM
```

---

## Effort Estimate

- **Complexity**: Very Low
- **Time**: 1-2 hours
- **Risk**: None

---

## References

- C++ Core Guidelines: ES.45 - Avoid "magic constants"; use symbolic constants
