# LightGBM Code Review - Issues Index

**Date**: November 22, 2025

---

## Summary

| Severity | Count | Categories |
|----------|-------|------------|
| High | 0 | - |
| Medium | 4 | Memory, Error Handling, Performance, API |
| Low | 4 | Code Style, API |

---

## Issues by Category

### Memory Management

| ID | Title | Severity | Status |
|----|-------|----------|--------|
| [ISSUE_001](memory_management/ISSUE_001_RAW_NEW_USAGE.md) | Raw `new` Usage Instead of `std::make_unique` | Medium | Open |

### Code Style

| ID | Title | Severity | Status |
|----|-------|----------|--------|
| [ISSUE_002](code_style/ISSUE_002_TYPEDEF_VS_USING.md) | Mixed Usage of `typedef` and `using` | Low | Open |
| [ISSUE_008](code_style/ISSUE_008_MAGIC_NUMBERS.md) | Magic Numbers in Code | Low | Open |

### Error Handling

| ID | Title | Severity | Status |
|----|-------|----------|--------|
| [ISSUE_003](error_handling/ISSUE_003_FIXED_ERROR_BUFFER.md) | Fixed-Size Error Message Buffer | Medium | Open |

### Performance

| ID | Title | Severity | Status |
|----|-------|----------|--------|
| [ISSUE_004](performance/ISSUE_004_REINTERPRET_CAST.md) | Unsafe Use of `reinterpret_cast` | Medium | Open |
| [ISSUE_007](performance/ISSUE_007_VECTOR_RESERVE.md) | Missing Vector Reserve Calls | Low | Open |

### API Design

| ID | Title | Severity | Status |
|----|-------|----------|--------|
| [ISSUE_005](api_design/ISSUE_005_MISSING_NOEXCEPT.md) | Missing `noexcept` Specifications | Medium | Open |
| [ISSUE_006](api_design/ISSUE_006_NODISCARD_MISSING.md) | Missing `[[nodiscard]]` Attribute | Low | Open |

---

## Priority Matrix

### Quick Wins (Low effort, High value)

1. **ISSUE_001**: Replace `new` with `make_unique` - Mechanical change, improves safety
2. **ISSUE_002**: Modernize typedefs - Purely cosmetic, improves consistency
3. **ISSUE_006**: Add `[[nodiscard]]` - Additive, catches bugs at compile time

### Medium Effort

4. **ISSUE_005**: Add `noexcept` - Requires audit, improves performance
5. **ISSUE_007**: Add `reserve()` calls - Requires profiling to prioritize
6. **ISSUE_003**: Increase error buffer - Simple fix, improves usability

### Requires Discussion

7. **ISSUE_004**: Replace `reinterpret_cast` - Design decision needed
8. **ISSUE_008**: Define constants - Need to agree on location and naming

---

## Estimated Total Effort

| Priority | Issues | Time Estimate |
|----------|--------|---------------|
| High | 0 | 0 hours |
| Medium | 4 | 8-12 hours |
| Low | 4 | 4-6 hours |
| **Total** | **8** | **12-18 hours** |

---

## Recommendations

1. **Start with ISSUE_001** (make_unique) - Highest impact, lowest risk
2. **Add [[nodiscard]]** (ISSUE_006) - Zero-risk, catches real bugs
3. **Address noexcept** (ISSUE_005) - Improves vector performance
4. **Schedule the rest** for regular maintenance

---

## Notes

- All issues are recommendations, not critical bugs
- The codebase is production-quality; these are improvements
- Python binding compatibility must be maintained for any API changes
- Some issues may be intentional design decisions (e.g., raw pointers for C API)
