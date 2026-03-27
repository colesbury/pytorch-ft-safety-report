# Free-Threading Safety Audit: jit/frontend (part 4)

**Overall Risk: Low**

## Files Reviewed
- `torch/csrc/jit/frontend/tree.h`
- `torch/csrc/jit/frontend/tree_views.cpp`
- `torch/csrc/jit/frontend/tree_views.h`
- `torch/csrc/jit/frontend/versioned_symbols.cpp`
- `torch/csrc/jit/frontend/versioned_symbols.h`

## Findings

### 1. Static const TreeList in Tree::trees() -- Safe
**File:** `tree.h`, line 47
```cpp
virtual const TreeList& trees() const {
    static const TreeList empty_trees = {};
    return empty_trees;
}
```
A `static const` object. Initialized once via constant initialization (empty vector) and never mutated. Safe under free-threading.

**Severity:** None

### 2. Static `symbol_range_map` -- Low Risk
**File:** `versioned_symbols.cpp`, line 63
```cpp
static std::unordered_map<Symbol, SymbolRange> symbol_range_map({...});
```
This is a file-scope static map initialized with constant data at static initialization time and never mutated after construction. Read-only concurrent access to an `unordered_map` is safe. Not a free-threading concern.

**Severity:** None

### 3. Static `kind_min_version_map` -- Safe
**File:** `versioned_symbols.cpp`, line 76
```cpp
static std::unordered_map<NodeKind, uint64_t> kind_min_version_map({...});
```
Same pattern as above: initialized once at static init time, never mutated, read-only access is safe.

**Severity:** None

### 4. `static SourceRange mergeRanges(...)` and `static inline` operator<< -- Safe
**File:** `tree.h`, lines 123, 213, 218
These are `static` functions in a header (internal linkage per TU). They operate on local/parameter data only. No shared mutable state.

**Severity:** None

## Summary

This group of files is essentially a tree data structure (AST nodes) and versioned symbol lookup tables. The tree types (`Tree`, `Compound`, `String`) are reference-counted value types created during parsing and not shared across threads. The versioned symbol maps are effectively immutable after static initialization. No Python C API usage, no global mutable state, no lazy initialization patterns of concern.

No changes are required for free-threading safety.
