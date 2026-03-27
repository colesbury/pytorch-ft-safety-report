# Free-Threading Safety Audit: jit_tensorexpr_operators__part1

## Files Audited
- `torch/csrc/jit/tensorexpr/operators/conv2d.cpp`
- `torch/csrc/jit/tensorexpr/operators/conv2d.h`
- `torch/csrc/jit/tensorexpr/operators/matmul.cpp`
- `torch/csrc/jit/tensorexpr/operators/matmul.h`
- `torch/csrc/jit/tensorexpr/operators/misc.cpp`
- `torch/csrc/jit/tensorexpr/operators/misc.h`
- `torch/csrc/jit/tensorexpr/operators/norm.cpp`
- `torch/csrc/jit/tensorexpr/operators/norm.h`
- `torch/csrc/jit/tensorexpr/operators/operators.h`
- `torch/csrc/jit/tensorexpr/operators/pointwise.cpp`
- `torch/csrc/jit/tensorexpr/operators/pointwise.h`
- `torch/csrc/jit/tensorexpr/operators/quantization.cpp`
- `torch/csrc/jit/tensorexpr/operators/quantization.h`
- `torch/csrc/jit/tensorexpr/operators/reduction.cpp`
- `torch/csrc/jit/tensorexpr/operators/reduction.h`
- `torch/csrc/jit/tensorexpr/operators/softmax.cpp`
- `torch/csrc/jit/tensorexpr/operators/softmax.h`

## Summary

This group contains the TensorExpr operator lowering implementations: conv2d, matmul, miscellaneous ops (cat, slice, broadcast, etc.), normalization, pointwise, quantization, reduction, and softmax. These files define how high-level operators are lowered to TensorExpr IR. They are all pure computational C++ code with no global mutable state and no Python C API usage.

## Issues Found

No thread-safety issues found.

## Concurrency Notes

- All files in this group contain only stateless helper functions and operator lowering implementations. Every function takes its inputs as parameters and returns computed IR expressions or tensors.
- Static helper functions are all pure: `_pair_int`, `_single_int_list` (conv2d.cpp), `squeezeIndices` (reduction.cpp), `checkTypes`, `isScalar`, `isOne`, `broadcastShapesImpl`, `processCatList`, `computeCatWoConditionals` (misc.cpp), `makeQBufHandle*`, `isChannelsLast`, `quant`, `dequant` (quantization.cpp). None modify global state.
- `operators.h` aggregates includes for all operator headers. No code. Safe.
- No Python C API calls (`PyObject*`, `PyGILState_Ensure`, etc.) in any of these files.
- No `static` mutable variables, no lazy initialization patterns, no global registries.
- This is the cleanest group in the entire audit -- fully safe for free-threaded execution.
