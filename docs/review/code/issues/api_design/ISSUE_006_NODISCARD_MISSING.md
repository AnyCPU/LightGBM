# Issue: Missing `[[nodiscard]]` Attribute

**Severity**: Low
**Category**: API Design / Safety
**Files Affected**: Factory functions, resource allocation functions

---

## Description

C++17 introduced the `[[nodiscard]]` attribute to indicate that a function's return value should not be ignored. LightGBM has many factory functions and resource allocation functions that would benefit from this attribute to prevent resource leaks and logic errors.

---

## Benefits of `[[nodiscard]]`

1. **Prevents resource leaks**: Compiler warns if return value is ignored
2. **Catches logic errors**: Ensures important results are used
3. **Self-documenting**: Indicates intent to callers
4. **Zero runtime cost**: Compile-time check only

---

## Affected Locations

### Example 1: Factory Functions

**File**: Multiple files in `src/`

```cpp
// Current - return value can be silently ignored
ObjectiveFunction* CreateObjectiveFunction(const std::string& type);
Metric* CreateMetric(const std::string& type);
TreeLearner* CreateTreeLearner(const std::string& type);

// Recommended
[[nodiscard]] std::unique_ptr<ObjectiveFunction>
CreateObjectiveFunction(const std::string& type);

[[nodiscard]] std::unique_ptr<Metric>
CreateMetric(const std::string& type);

[[nodiscard]] std::unique_ptr<TreeLearner>
CreateTreeLearner(const std::string& type);
```

### Example 2: Resource Allocation

**File**: Various headers

```cpp
// Current
Dataset* CreateDatasetFromFile(const char* filename);

// Recommended
[[nodiscard]] DatasetHandle CreateDatasetFromFile(const char* filename);
```

### Example 3: Status/Result Functions

```cpp
// Current - easy to ignore error check
int LGBM_BoosterCreate(...);

// Recommended (via macro for C compatibility)
#ifdef __cplusplus
  #define LGBM_NODISCARD [[nodiscard]]
#else
  #define LGBM_NODISCARD
#endif

LGBM_NODISCARD LIGHTGBM_C_EXPORT int LGBM_BoosterCreate(...);
```

### Example 4: Getter Functions with Computation

```cpp
// Current - computation result might be ignored
std::vector<double> GetPredictions() const;
std::string SaveModelToString() const;

// Recommended
[[nodiscard]] std::vector<double> GetPredictions() const;
[[nodiscard]] std::string SaveModelToString() const;
```

---

## Guidelines for `[[nodiscard]]`

| Function Type | Use `[[nodiscard]]` | Reason |
|---------------|---------------------|--------|
| Factory functions | Yes | Prevents resource leak |
| Allocation functions | Yes | Prevents memory leak |
| Functions returning error codes | Yes | Prevents ignored errors |
| Expensive computations | Yes | Prevents wasted work |
| Pure functions | Yes | Result is only purpose |
| Getters with side effects | No | Side effect may be intent |
| void functions | N/A | No return value |

---

## Recommended Changes

### 1. Add to Factory Functions

```cpp
// File: include/LightGBM/objective_function.h
class ObjectiveFunction {
 public:
  [[nodiscard]] static std::unique_ptr<ObjectiveFunction>
  Create(const std::string& type, const Config& config);
};

// File: include/LightGBM/metric.h
class Metric {
 public:
  [[nodiscard]] static std::unique_ptr<Metric>
  Create(const std::string& type, const Config& config);
};
```

### 2. Add to C API Functions

```cpp
// File: include/LightGBM/c_api.h
// Define portable nodiscard macro
#if defined(__cplusplus) && __cplusplus >= 201703L
  #define LGBM_NODISCARD [[nodiscard]]
#elif defined(__has_cpp_attribute)
  #if __has_cpp_attribute(nodiscard)
    #define LGBM_NODISCARD [[nodiscard]]
  #else
    #define LGBM_NODISCARD
  #endif
#else
  #define LGBM_NODISCARD
#endif

// Apply to functions returning status codes
LGBM_NODISCARD LIGHTGBM_C_EXPORT int LGBM_BoosterCreate(...);
LGBM_NODISCARD LIGHTGBM_C_EXPORT int LGBM_DatasetCreateFromFile(...);
```

### 3. Add to Resource-Returning Functions

```cpp
// File: include/LightGBM/boosting.h
class Boosting {
 public:
  [[nodiscard]] virtual std::string SaveModelToString() const = 0;
  [[nodiscard]] virtual std::vector<double> FeatureImportance() const = 0;
  [[nodiscard]] virtual double GetUpperBoundValue() const = 0;
};
```

---

## Example: Bug Prevention

```cpp
// Without [[nodiscard]], this bug compiles silently:
void ProcessData() {
    LGBM_BoosterCreate(dataset, params, &booster);  // Error code ignored!
    // booster might be invalid, but we don't know
}

// With [[nodiscard]]:
void ProcessData() {
    LGBM_BoosterCreate(dataset, params, &booster);  // Warning: ignoring return value
    // Compiler catches the bug
}

// Correct usage:
void ProcessData() {
    int result = LGBM_BoosterCreate(dataset, params, &booster);
    if (result != 0) {
        // Handle error
    }
}
```

---

## Impact Analysis

| Aspect | Before | After |
|--------|--------|-------|
| Compile warnings | None | Warns on ignored values |
| Runtime behavior | Unchanged | Unchanged |
| Source compatibility | N/A | Backward compatible |
| Binary compatibility | N/A | Unchanged |

---

## Effort Estimate

- **Complexity**: Very Low
- **Time**: 1-2 hours
- **Risk**: None (purely additive, warnings only)

---

## References

- C++ Core Guidelines: F.21 - To return multiple "out" values, prefer returning a struct
- C++ attribute: nodiscard - https://en.cppreference.com/w/cpp/language/attributes/nodiscard
- P0189R1 - [[nodiscard]] attribute
