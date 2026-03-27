# Free-Threading Safety Audit: jit_passes_onnx__part2

## Overall Risk: LOW

## Files Audited
- `jit/passes/onnx/function_extraction.h`
- `jit/passes/onnx/function_substitution.cpp`
- `jit/passes/onnx/function_substitution.h`
- `jit/passes/onnx/helper.cpp`
- `jit/passes/onnx/helper.h`
- `jit/passes/onnx/list_model_parameters.cpp`
- `jit/passes/onnx/list_model_parameters.h`
- `jit/passes/onnx/naming.cpp`
- `jit/passes/onnx/naming.h`
- `jit/passes/onnx/onnx_log.cpp`
- `jit/passes/onnx/onnx_log.h`
- `jit/passes/onnx/peephole.cpp`
- `jit/passes/onnx/peephole.h`
- `jit/passes/onnx/prepare_division_for_onnx.cpp`
- `jit/passes/onnx/prepare_division_for_onnx.h`

## Summary

Most files in this group are pure graph transformation passes operating on local state. The header `helper.h` defines `static const int` opset version constants, which are immutable and safe. The `peephole.cpp` has some function-local static data structures that are `const` or only read.

## Detailed Analysis

### Issues Found

#### Issue 1: Static `broadcast_positions` map in peephole.cpp (line 72) -- LOW

```cpp
static std::unordered_map<NodeKind, std::vector<size_t>> broadcast_positions = {
    ...
};
```

This is a function-local static mutable map (inside `getBroadcastPositions()`). It is populated with constant data at initialization and never modified. C++11 guarantees thread-safe initialization. However, it should be declared `const` for clarity.

### No Issues

- **function_extraction.h**: Pure declarations.
- **function_substitution.cpp/h**: Local graph processing.
- **helper.cpp/h**: Utility functions; header has `static const int` constants. No mutable statics.
- **list_model_parameters.cpp**: Local functions with no shared state.
- **naming.cpp/h**: Utility functions for ONNX naming.
- **onnx_log.cpp/h**: Logging utilities.
- **prepare_division_for_onnx.cpp**: Pure graph traversal.

## Recommendations

1. **peephole.cpp**: Mark `broadcast_positions` as `const` since it is never modified after initialization.
