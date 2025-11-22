# Issue: Missing Vector Reserve Calls

**Severity**: Low-Medium
**Category**: Performance
**Files Affected**: Various implementation files

---

## Description

Several code paths in LightGBM construct vectors in loops without calling `reserve()` first. This can lead to multiple reallocations and copies, impacting performance especially for large datasets.

---

## Pattern to Watch For

```cpp
// Suboptimal - may reallocate multiple times
std::vector<double> results;
for (size_t i = 0; i < n; ++i) {
    results.push_back(ComputeValue(i));  // May trigger reallocation
}

// Optimal - single allocation
std::vector<double> results;
results.reserve(n);  // Pre-allocate
for (size_t i = 0; i < n; ++i) {
    results.push_back(ComputeValue(i));  // No reallocation
}
```

---

## Performance Impact

| Initial Size | Insertions | Allocations (no reserve) | Allocations (with reserve) |
|--------------|------------|--------------------------|----------------------------|
| 0 | 100 | ~7 | 1 |
| 0 | 1,000 | ~10 | 1 |
| 0 | 1,000,000 | ~20 | 1 |

Each reallocation copies all existing elements, leading to O(n) extra work per reallocation.

---

## Recommended Pattern

```cpp
// Pattern 1: Known size
std::vector<Result> results;
results.reserve(known_count);
for (size_t i = 0; i < known_count; ++i) {
    results.push_back(Compute(i));
}

// Pattern 2: Transform with std::transform (C++17 back_inserter)
std::vector<double> results;
results.reserve(inputs.size());
std::transform(inputs.begin(), inputs.end(),
               std::back_inserter(results),
               [](const auto& input) { return Transform(input); });

// Pattern 3: Use resize + index assignment for POD types
std::vector<double> results(n);  // Default-initialized
for (size_t i = 0; i < n; ++i) {
    results[i] = Compute(i);
}
```

---

## Audit Recommendation

Run the following grep to find potential optimization sites:

```bash
# Find push_back in loops without prior reserve
grep -n "push_back" src/*.cpp src/**/*.cpp | head -50
```

Review each occurrence to check if `reserve()` would be beneficial.

---

## Effort Estimate

- **Complexity**: Low
- **Time**: 2-3 hours (audit and fix)
- **Risk**: Very low

---

## References

- C++ Core Guidelines: Per.1 - Don't optimize without reason
- C++ Core Guidelines: Per.11 - Move computation from run time to compile time
