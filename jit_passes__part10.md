# Free-Threading Safety Audit: jit_passes__part10

## Overall Risk: MEDIUM

## Files Audited
- `jit/passes/remove_expands.cpp`
- `jit/passes/remove_expands.h`
- `jit/passes/remove_inplace_ops.cpp`
- `jit/passes/remove_inplace_ops.h`
- `jit/passes/remove_mutation.cpp`
- `jit/passes/remove_mutation.h`
- `jit/passes/remove_redundant_profiles.cpp`
- `jit/passes/remove_redundant_profiles.h`
- `jit/passes/replacement_of_old_operators.cpp`
- `jit/passes/replacement_of_old_operators.h`
- `jit/passes/requires_grad_analysis.cpp`
- `jit/passes/requires_grad_analysis.h`
- `jit/passes/restore_mutation.cpp`
- `jit/passes/restore_mutation.h`
- `jit/passes/shape_analysis.cpp`

## Summary

Most files are pure graph transformation passes with no shared mutable state. However, `shape_analysis.cpp` contains a significant static mutable state pattern that poses a race condition risk under free-threading.

## Detailed Analysis

### Issues Found

#### Issue 1: Static mutable `shape_formulas` vector and mutex (shape_analysis.cpp, lines 790-796) -- MEDIUM

```cpp
static std::mutex shape_formulas_mutex;
static std::vector<std::pair<OperatorSet, formula_t>> shape_formulas;
struct register_formula_for {
    register_formula_for(OperatorSet operators, formula_t formula) {
        std::unique_lock<std::mutex> lock{shape_formulas_mutex};
        shape_formulas.emplace_back(std::move(operators), std::move(formula));
    }
};
```

The `shape_formulas` vector is populated by static `register_formula_for` objects constructed during static initialization (lines 807-1561). The writes are mutex-protected. However, the reads at line 1565 (`for (auto& entry : shape_formulas)`) occur without holding the mutex. This is safe only if all static registrations complete before any reads, which is guaranteed during normal single-threaded static init. But under free-threading, if `PropagateInputShapes` is called from a Python thread while dynamic library loading triggers additional static initializers in another thread, this would be a data race. In practice, this is low risk because these registrations happen during the loading of this translation unit.

#### Issue 2: Static mutable `resize_ops` set (shape_analysis.cpp, line 259) -- LOW

```cpp
static std::unordered_set<Symbol> resize_ops{...};
```

This is inside a member function `resizesInput()`, constructed once via function-local static initialization. C++11 guarantees thread-safe initialization of function-local statics, so this is safe.

#### Issue 3: Static `OperatorSet comparison_ops` (requires_grad_analysis.cpp, line 45) -- LOW

```cpp
static const OperatorSet comparison_ops = {...};
```

Function-local `const` static. Thread-safe initialization guaranteed by C++11.

#### Issue 4: `activation_ops` static set (remove_mutation.cpp, lines 365-372) -- LOW

```cpp
static const std::unordered_set<Symbol> activation_ops = []() {
    std::unordered_set<Symbol> target_ops;
    for (const auto& iter : activation_type_promotion_mapping) { ... }
    return target_ops;
}();
```

File-scope static with `const`. Initialized once at program startup, then only read. Safe.

#### Issue 5: `activation_type_promotion_mapping` global const (restore_mutation.h, line 12) -- LOW

```cpp
const std::unordered_map<Symbol, bool> activation_type_promotion_mapping = {...};
```

Header-level `const` global. This is initialized during static initialization. Since it is declared `const`, it is read-only after construction. However, being in a header, it gets a copy per translation unit, which is wasteful but not a thread-safety issue.

#### Issue 6: Static `inPlaceToOutOfPlace` and `expectedInputCount` maps (remove_inplace_ops.cpp, lines 6-21) -- LOW

These are `static const` maps inside an anonymous namespace. Immutable after initialization.

### No Issues

- **remove_expands.cpp**: Pure graph traversal, no shared state.
- **remove_mutation.cpp/h**: `MutationRemover` is an instance-local class. The `getOrCreateAliasDb()` pattern is per-instance lazy init, not global.
- **remove_redundant_profiles.cpp**: Pure graph traversal with local `AliasDb`.
- **replacement_of_old_operators.cpp**: Local `OldOpsReplacerWithUpgraders` struct. References `get_operator_version_map()` and `dump_upgraders_map()` which are defined elsewhere.
- **restore_mutation.cpp**: Instance-local `FunctionalToInplaceRewriter`.

## Recommendations

1. **shape_analysis.cpp**: Consider adding a read-lock or `std::shared_mutex` for the `shape_formulas` iteration at line 1565 if there is any possibility of dynamic registration after startup. Alternatively, verify that all `register_formula_for` static objects are fully constructed before any thread calls the shape analysis pass, which is likely already guaranteed by normal startup ordering.
