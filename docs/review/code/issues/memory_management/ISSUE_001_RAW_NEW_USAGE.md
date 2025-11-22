# Issue: Raw `new` Usage Instead of `std::make_unique`

**Severity**: Medium
**Category**: Memory Management / Modern C++
**Files Affected**: Multiple (31+ files with 209 instances)

---

## Description

The codebase contains 209 instances of raw `new` usage across 31 files. While many of these are wrapped in `unique_ptr` immediately after allocation, using `std::make_unique` is preferred for several reasons:

1. **Exception safety**: `make_unique` provides strong exception guarantee
2. **Conciseness**: Single expression instead of two
3. **Consistency**: Modern C++ style
4. **Performance**: One allocation instead of potentially two (for shared_ptr)

---

## Affected Locations

### Example 1: Tree Creation in GBDT

**File**: `/home/user/LightGBM/src/boosting/gbdt.h`
**Lines**: 77, 84, 110

```cpp
// Current code (line 77)
auto new_tree = std::unique_ptr<Tree>(new Tree(*(tree.get())));
models_.push_back(std::move(new_tree));
```

**Recommended:**
```cpp
models_.push_back(std::make_unique<Tree>(*tree));
```

### Example 2: Factory Pattern Returns

**File**: `/home/user/LightGBM/src/metric/metric.cpp`
Multiple factories return `new` objects:

```cpp
// Current
return new BinaryLoglossMetric(config);

// Recommended
return std::make_unique<BinaryLoglossMetric>(config);
```

---

## Impact Analysis

| Aspect | Current Risk | After Fix |
|--------|--------------|-----------|
| Memory leaks | Low (usually wrapped in unique_ptr) | None |
| Exception safety | Medium | Strong guarantee |
| Code readability | Medium | Improved |
| Compilation errors | None | None |

---

## Recommended Fix

### Step 1: Replace obvious cases

Use regex replacement in IDEs:
- Find: `std::unique_ptr<(\w+)>\(new (\w+)\(`
- Replace: `std::make_unique<$1>(`

### Step 2: Handle factory functions

Factory functions should return `unique_ptr`:

```cpp
// Before
ObjectiveFunction* CreateObjectiveFunction(const std::string& type) {
  if (type == "binary") {
    return new BinaryLogloss(config);
  }
  // ...
}

// After
std::unique_ptr<ObjectiveFunction> CreateObjectiveFunction(const std::string& type) {
  if (type == "binary") {
    return std::make_unique<BinaryLogloss>(config);
  }
  // ...
}
```

### Step 3: Handle C API compatibility

For C API functions that need raw pointers:

```cpp
// Internal implementation uses unique_ptr
auto booster = std::make_unique<Booster>(config);

// Only expose raw pointer at API boundary
*out = booster.release();
```

---

## Testing Requirements

1. Ensure all existing tests pass
2. Add valgrind/ASAN testing to verify no memory leaks
3. Test exception scenarios to verify proper cleanup

---

## Effort Estimate

- **Complexity**: Low
- **Time**: 2-4 hours
- **Risk**: Low (mostly mechanical changes)

---

## References

- C++ Core Guidelines: R.11 - Avoid calling `new` and `delete` explicitly
- C++ Core Guidelines: R.23 - Use `make_unique` to make `unique_ptr`
- https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#Rr-make_unique
