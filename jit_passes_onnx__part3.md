# Free-Threading Safety Audit: jit_passes_onnx__part3

## Overall Risk: MEDIUM

## Files Audited
- `jit/passes/onnx/preprocess_for_onnx.cpp`
- `jit/passes/onnx/preprocess_for_onnx.h`
- `jit/passes/onnx/remove_inplace_ops_for_onnx.cpp`
- `jit/passes/onnx/remove_inplace_ops_for_onnx.h`
- `jit/passes/onnx/scalar_type_analysis.cpp`
- `jit/passes/onnx/scalar_type_analysis.h`
- `jit/passes/onnx/shape_type_inference.cpp`
- `jit/passes/onnx/shape_type_inference.h`
- `jit/passes/onnx/unpack_quantized_weights.cpp`
- `jit/passes/onnx/unpack_quantized_weights.h`

## Summary

The main concern is `shape_type_inference.cpp`, which uses the Python C API to access PyObject structures (PyTuple, PyList, PyDict) with borrowed references, uses the global `ConstantValueMap` singleton (see jit_passes_onnx__part1 report), and has static mutable data structures. The `scalar_type_analysis.cpp` has static mutable data that should be `const`.

## Detailed Analysis

### Issues Found

#### Issue 1: Python C API usage without critical sections (shape_type_inference.cpp, lines 2287-2474) -- MEDIUM

```cpp
static size_t ONNXAssignOutputShape(
    std::shared_ptr<Graph>& graph,
    size_t outputs_index,
    PyObject* output_obj, ...) {
    ...
    if (PyTuple_Check(output_obj)) {
        size_t tuple_len = PyTuple_GET_SIZE(output_obj);
        for (...) {
            ONNXAssignOutputShape(..., PyTuple_GET_ITEM(output_obj, i), ...);
        }
    } else if (PyList_Check(output_obj)) {
        ...PyList_GET_ITEM(output_obj, i)...
    } else if (PyDict_Check(output_obj)) {
        auto* items = PyDict_Items(output_obj);
        ...
        Py_DECREF(items);
    }
}
```

This function accesses Python containers (tuples, lists, dicts) using borrowed references (`PyTuple_GET_ITEM`, `PyList_GET_ITEM`) without `Py_BEGIN_CRITICAL_SECTION`. Under free-threading, another thread could modify these containers concurrently, invalidating the borrowed references. The `PyDict_Items` call also returns a new reference that could become stale.

The caller `ONNXAssignOutputShape` (line 2451) also calls `unflatten()` which returns a `PyObject*` that is later `Py_DECREF`'d -- this must be protected from concurrent access.

#### Issue 2: Static mutable maps in shape_type_inference.cpp -- LOW

```cpp
static std::unordered_map<std::string, std::unordered_set<int64_t>>
    non_required_shape_inference_idx_map = {{"onnx::LSTM", {4}}};  // line 1915

static std::unordered_set<std::string> nodeTypeReliableForTracer = {...};  // line 1974
```

These are initialized with constant data and never modified. They should be `const`.

#### Issue 3: Heavy use of `ConstantValueMap` singleton (shape_type_inference.cpp, throughout) -- MEDIUM

The file extensively uses `ConstantValueMap::SetShape`, `GetShape`, `SetTypeReliable`, `ClearMaps`, etc. All of these operate on the unsynchronized global singleton (analyzed in jit_passes_onnx__part1). The function `ONNXShapeTypeInference` calls `ClearMaps()` at start and end, which means concurrent calls would corrupt each other's state.

#### Issue 4: Static mutable maps in scalar_type_analysis.cpp -- LOW

```cpp
static const std::unordered_map<c10::ScalarType, int, ScalarTypeHashFunction>
    scalarTypeToONNXTypeMap = {...};  // line 16
static const std::unordered_set<NodeKind> standardOps = {...};  // line 48
static const std::unordered_set<NodeKind> comparisonOps = {...};  // line 65
static const std::unordered_set<NodeKind> selectorOps = {...};  // line 73
```

These are all `const`. Safe.

### No Issues

- **preprocess_for_onnx.cpp**: Pure local graph processing.
- **remove_inplace_ops_for_onnx.cpp**: Local graph processing using `MutationRemover`.
- **unpack_quantized_weights.cpp**: Local graph processing; uses `Dispatcher::singleton()` which is externally managed.

## Recommendations

1. **shape_type_inference.cpp**: Wrap the `ONNXAssignOutputShape` function's Python container accesses with `Py_BEGIN_CRITICAL_SECTION`/`Py_END_CRITICAL_SECTION` on the relevant PyObject. Under free-threading, borrowed references from Python containers are unsafe without critical sections.
2. **shape_type_inference.cpp**: Mark `non_required_shape_inference_idx_map` and `nodeTypeReliableForTracer` as `const`.
3. The `ConstantValueMap` issue (singleton without synchronization) should be addressed globally -- see jit_passes_onnx__part1 recommendations.
