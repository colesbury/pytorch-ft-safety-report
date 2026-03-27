# Free-Threading Safety Audit: jit_passes__part2

**Overall Risk: LOW**

All files in this group are pure C++ JIT graph transformations. There is no Python C API usage. The only notable item is a static mutable pointer in `device_type_analysis.cpp` that implements a lazily-initialized singleton registry.

---

## File Analysis

### jit/passes/check_strict_fusion.h
- **Risk: NONE**
- Header-only declaration.

### jit/passes/clear_profiling.cpp, .h
- **Risk: NONE**
- Pure graph transformation. Iterates blocks and resets type information on tensor outputs. All state is stack-local.

### jit/passes/clear_undefinedness.cpp, .h
- **Risk: NONE**
- Pure graph transformation. Replaces tensor types with `TensorType::get()`. All state is stack-local.

### jit/passes/common_subexpression_elimination.cpp, .h
- **Risk: NONE**
- The `CommonSubexpressionEliminator` struct is created per-invocation, holding a graph and a lazily-initialized `AliasDb`. No static/global mutable state. No Python C API.

### jit/passes/concat_opt.cpp, .h
- **Risk: NONE**
- Three pass classes (`ConcatCommonInputsEliminator`, `ConcatExpander`, `ConcatCombiner`) are all created per-invocation with local state. No static/global mutable state. No Python C API.

### jit/passes/constant_pooling.cpp, .h
- **Risk: NONE**
- Pure graph transformation. Deduplicates constants. All state (the `constants` set, `AliasDb`) is created per-call. No static/global mutable state.

### jit/passes/constant_propagation.cpp, .h
- **Risk: LOW**
- `static std::unordered_set<Symbol> skip_list = {...}` (line 132): This is initialized once at static init time and never modified afterward. It is effectively immutable after initialization. Safe for concurrent reads.
- The `ConstantPropagator` struct is created per-invocation with all state local.
- `runNodeIfInputsAreConstant` calls `n->getOperation()` and runs the op on a local `Stack`. This is purely C++ and does not interact with the Python runtime.

### jit/passes/create_autodiff_subgraphs.cpp, .h
- **Risk: NONE**
- The `SubgraphSlicer` class and all helper structures (`ContextMapping`, etc.) are created per-invocation with local state. No static/global mutable state. No Python C API.

---

## Summary of Issues

| File | Issue | Severity | Description |
|------|-------|----------|-------------|
| constant_propagation.cpp | Static `skip_list` | None | Initialized at static init, never modified. Read-only after init. Safe. |

## Recommendations

No changes needed for free-threading safety in this group. All passes operate on local data structures created per-invocation, and the single static set is effectively immutable.
