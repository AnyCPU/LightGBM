# Issue: Missing `noexcept` Specifications

**Severity**: Medium
**Category**: API Design / Performance
**Files Affected**: Multiple header files

---

## Description

The codebase is missing `noexcept` specifications on many functions that should not throw exceptions. This is particularly important for:

1. **Destructors**: Should always be `noexcept` (implicitly `noexcept` in C++11+, but explicit is clearer)
2. **Move constructors/assignment**: Required for `std::vector` optimization
3. **Swap operations**: STL algorithms require noexcept swap
4. **Simple getters**: Improve documentation of exception safety

---

## Impact

Without `noexcept`:

1. **Performance**: `std::vector` uses copy instead of move when resizing
2. **Documentation**: Users don't know what exceptions to expect
3. **Optimization**: Compiler can't optimize for no-throw paths

---

## Affected Locations

### Example 1: GBDT Class

**File**: `/home/user/LightGBM/src/boosting/gbdt.h`
**Lines**: 42-47

```cpp
// Current
GBDT();
~GBDT();

// Recommended
GBDT();
~GBDT() noexcept;  // Destructor should never throw
```

### Example 2: Move Operations

If move operations exist, they should be `noexcept`:

```cpp
// Current (if exists)
Tree(Tree&& other);
Tree& operator=(Tree&& other);

// Recommended
Tree(Tree&& other) noexcept;
Tree& operator=(Tree&& other) noexcept;
```

### Example 3: Simple Getters

**File**: `/home/user/LightGBM/src/boosting/gbdt.h`
**Lines**: 164, 394, 406, 412, 418, 424

```cpp
// Current
int GetCurrentIteration() const override { return ...; }
inline int MaxFeatureIdx() const override { return max_feature_idx_; }
inline int LabelIdx() const override { return label_idx_; }

// Recommended
int GetCurrentIteration() const noexcept override { return ...; }
inline int MaxFeatureIdx() const noexcept override { return max_feature_idx_; }
inline int LabelIdx() const noexcept override { return label_idx_; }
```

---

## Guidelines for `noexcept`

| Function Type | Use `noexcept` | Notes |
|---------------|----------------|-------|
| Destructors | Always | Never throw from destructors |
| Move constructor | Yes | Required for vector optimization |
| Move assignment | Yes | Required for vector optimization |
| Swap | Yes | STL requires noexcept swap |
| Simple getters | Yes | Document no-throw guarantee |
| Functions that allocate | Usually no | `bad_alloc` possible |
| Functions calling C API | Conditional | `noexcept(false)` if error possible |

---

## Recommended Changes

### 1. Destructors

```cpp
// All destructors should be explicitly noexcept
class GBDT : public GBDTBase {
 public:
  ~GBDT() noexcept override;
};

class Tree {
 public:
  ~Tree() noexcept;
};

class Dataset {
 public:
  ~Dataset() noexcept;
};
```

### 2. Move Operations

```cpp
// Add move operations with noexcept where missing
class Tree {
 public:
  Tree(Tree&& other) noexcept;
  Tree& operator=(Tree&& other) noexcept;

  // If move is complex and might throw, at least document it
  // Tree(Tree&& other) noexcept(false);  // May throw due to...
};
```

### 3. Simple Getters in Interfaces

```cpp
// File: include/LightGBM/boosting.h
class Boosting {
 public:
  virtual int GetCurrentIteration() const noexcept = 0;
  virtual int MaxFeatureIdx() const noexcept = 0;
  virtual int NumberOfClasses() const noexcept = 0;
  // ...
};
```

### 4. Conditional noexcept

For functions that may or may not throw depending on operations:

```cpp
template<typename T>
void swap(T& a, T& b) noexcept(noexcept(std::swap(a, b))) {
    std::swap(a, b);
}
```

---

## Vector Optimization Demonstration

```cpp
#include <vector>
#include <iostream>

struct WithNoexcept {
    WithNoexcept(WithNoexcept&&) noexcept { std::cout << "Move\n"; }
};

struct WithoutNoexcept {
    WithoutNoexcept(WithoutNoexcept&&) { std::cout << "Move\n"; }
};

int main() {
    std::vector<WithNoexcept> v1;
    v1.reserve(1);
    v1.emplace_back();
    v1.emplace_back();  // Triggers reallocation - uses MOVE

    std::vector<WithoutNoexcept> v2;
    v2.reserve(1);
    v2.emplace_back();
    v2.emplace_back();  // Triggers reallocation - uses COPY!
}
```

---

## Impact Analysis

| Change | Performance Impact | Safety Impact |
|--------|-------------------|---------------|
| Destructors | None | Documentation |
| Move operations | Significant (vector resize) | High |
| Getters | Minor | Documentation |

---

## Effort Estimate

- **Complexity**: Low-Medium
- **Time**: 4-6 hours (audit all classes)
- **Risk**: Very low (adding noexcept is additive)

---

## References

- C++ Core Guidelines: E.12 - Use `noexcept` when exiting a function because of a `throw` is impossible or unacceptable
- C++ Core Guidelines: F.6 - If your function must not throw, declare it `noexcept`
- https://en.cppreference.com/w/cpp/language/noexcept_spec
