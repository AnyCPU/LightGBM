# Issue: Mixed Usage of `typedef` and `using`

**Severity**: Low
**Category**: Code Style / Consistency
**Files Affected**: Multiple header files

---

## Description

The codebase uses both legacy C-style `typedef` and modern C++ `using` declarations inconsistently. While both are functionally equivalent for simple type aliases, `using` is preferred in modern C++ for:

1. Consistency with template aliases (where `typedef` cannot be used)
2. More readable syntax
3. Better tooling support

---

## Affected Locations

### Example 1: Core Type Definitions

**File**: `/home/user/LightGBM/include/LightGBM/meta.h`
**Lines**: 28-31

```cpp
// Current code - using typedef
typedef int32_t data_size_t;
typedef float label_t;
typedef float score_t;
```

**Recommended:**
```cpp
// Modern style - using declarations
using data_size_t = int32_t;
using label_t = float;
using score_t = float;
```

### Example 2: C API Types

**File**: `/home/user/LightGBM/include/LightGBM/c_api.h`
**Lines**: 30-33

```cpp
// Current code - typedef for C compatibility
typedef void* DatasetHandle;
typedef void* BoosterHandle;
typedef void* FastConfigHandle;
typedef void* ByteBufferHandle;
```

**Note**: For C API headers, `typedef` is actually correct as these need C compatibility.

### Example 3: Mixed Usage in Same File

**File**: `/home/user/LightGBM/include/LightGBM/meta.h`

```cpp
// Line 28 - legacy typedef
typedef int32_t data_size_t;

// Line 42 - modern using
using PredictFunction = std::function<void(const std::vector<std::pair<int, double>>&, double* output)>;
```

---

## Guidelines

| Context | Recommended Style | Reason |
|---------|-------------------|--------|
| C headers (extern "C") | `typedef` | C compatibility |
| C++ headers | `using` | Modern style |
| Template aliases | `using` | `typedef` doesn't support |

---

## Recommended Fix

### C++ Headers Only

Change all `typedef` in C++ code to `using`:

```cpp
// Before
typedef int32_t data_size_t;
typedef float label_t;
typedef std::vector<double> ScoreVector;

// After
using data_size_t = int32_t;
using label_t = float;
using ScoreVector = std::vector<double>;
```

### Preserve C API Compatibility

Keep `typedef` in `c_api.h` which needs C compatibility:

```cpp
// Keep typedef for C API
typedef void* DatasetHandle;  // OK - needed for C compatibility
```

---

## Impact Analysis

| Aspect | Risk | Notes |
|--------|------|-------|
| Binary compatibility | None | Same ABI |
| Source compatibility | None | Both syntaxes accepted |
| Build time | None | Equivalent compilation |
| Readability | Improved | More consistent |

---

## Effort Estimate

- **Complexity**: Very Low
- **Time**: 1-2 hours
- **Risk**: None (purely cosmetic)

---

## References

- C++ Core Guidelines: T.43 - Prefer `using` over `typedef` for defining aliases
- https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#Rt-using
