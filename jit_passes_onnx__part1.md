# Free-Threading Safety Audit: jit_passes_onnx__part1

## Overall Risk: MEDIUM

## Files Audited
- `jit/passes/onnx/cast_all_constant_to_floating.cpp`
- `jit/passes/onnx/cast_all_constant_to_floating.h`
- `jit/passes/onnx/constant_fold.cpp`
- `jit/passes/onnx/constant_fold.h`
- `jit/passes/onnx/constant_map.cpp`
- `jit/passes/onnx/constant_map.h`
- `jit/passes/onnx/deduplicate_initializers.cpp`
- `jit/passes/onnx/deduplicate_initializers.h`
- `jit/passes/onnx/eliminate_unused_items.cpp`
- `jit/passes/onnx/eliminate_unused_items.h`
- `jit/passes/onnx/eval_peephole.cpp`
- `jit/passes/onnx/eval_peephole.h`
- `jit/passes/onnx/fixup_onnx_controlflow.cpp`
- `jit/passes/onnx/fixup_onnx_controlflow.h`
- `jit/passes/onnx/function_extraction.cpp`

## Summary

The primary concern is the `ConstantValueMap` singleton and the global mutable state in `function_extraction.cpp`. The `ConstantValueMap` is a classic Meyer's singleton with zero synchronization, storing and accessing multiple mutable hash maps from static methods. The ONNX export path is generally single-threaded (runs under GIL in Python callers), but under free-threading these become data races.

## Detailed Analysis

### Issues Found

#### Issue 1: `ConstantValueMap` singleton with unsynchronized mutable state (constant_map.cpp, line 12) -- MEDIUM

```cpp
ConstantValueMap& ConstantValueMap::getInstance() {
    static ConstantValueMap s;
    return s;
}
```

The singleton itself is thread-safe to construct (C++11 guarantee), but all subsequent accesses to the internal maps (`rankMap`, `shapeMap`, `tensorValueMap`, `typeReliableMap`, `useInferredTypeMap`, `shapeValueMap`, `inferredShapeData`, `symbolDimMap`, `dimSymbolMap`) are completely unsynchronized. Every `SetRank`, `GetRank`, `HasRank`, `SetShape`, `GetShape`, `SetValue`, `ClearMaps`, etc. performs read-modify-write on shared mutable state through the singleton without any locking.

In practice, the ONNX export pipeline is invoked from Python and runs sequentially, with `ClearMaps()` called at the start and end. However, if two threads attempted ONNX export simultaneously under free-threading, this would be a data race.

#### Issue 2: Global mutable state in function_extraction.cpp (lines 16-17) -- MEDIUM

```cpp
static std::unordered_map<ScopePtr, Node*> scope_attr_map_;
static std::shared_ptr<Graph> scope_attr_graph_ = std::make_shared<Graph>();
```

File-scope mutable globals. These are written and read during ONNX function extraction without any synchronization. Concurrent ONNX exports would race on these.

#### Issue 3: Static mutable `onnxTypeToScalarTypeMap` (constant_fold.cpp, line 33) -- LOW

```cpp
static std::unordered_map<int, at::ScalarType> onnxTypeToScalarTypeMap = {...};
```

Although declared as `static` (non-const) mutable, this map is initialized at file scope with constant values and never modified thereafter. Effectively immutable. Should be declared `const` for clarity.

### No Issues

- **cast_all_constant_to_floating.cpp**: Pure graph traversal.
- **deduplicate_initializers.cpp**: Pure local graph processing.
- **eliminate_unused_items.cpp**: Pure local graph processing.
- **eval_peephole.cpp**: Local functions operating on passed-in data.
- **fixup_onnx_controlflow.cpp**: Local functions operating on graph nodes.

## Recommendations

1. **constant_map.cpp**: If concurrent ONNX exports become possible, the `ConstantValueMap` singleton needs synchronization (e.g., a mutex wrapping all map accesses), or it should be redesigned to use per-export-session state rather than a global singleton.
2. **function_extraction.cpp**: Move `scope_attr_map_` and `scope_attr_graph_` from file-scope globals into a local context object passed through the extraction pipeline.
3. **constant_fold.cpp**: Mark `onnxTypeToScalarTypeMap` as `const`.
