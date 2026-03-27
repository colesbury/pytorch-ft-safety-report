# Free-Threading Audit: autograd_functions

**Files:**
- `torch/csrc/autograd/functions/accumulate_grad.cpp`
- `torch/csrc/autograd/functions/accumulate_grad.h`
- `torch/csrc/autograd/functions/basic_ops.cpp`
- `torch/csrc/autograd/functions/basic_ops.h`
- `torch/csrc/autograd/functions/comm.cpp`
- `torch/csrc/autograd/functions/comm.h`
- `torch/csrc/autograd/functions/init.cpp`
- `torch/csrc/autograd/functions/pybind.h`
- `torch/csrc/autograd/functions/tensor.cpp`
- `torch/csrc/autograd/functions/tensor.h`
- `torch/csrc/autograd/functions/utils.cpp`
- `torch/csrc/autograd/functions/utils.h`

**Overall Risk:** None

## Issues

No free-threading issues found.

## Summary

These files are largely free of free-threading concerns. The core autograd node implementations (`AccumulateGrad`, `CopySlices`, `CopyBackwards`, etc.) operate on per-node instance state protected by per-node mutexes where needed (e.g., `AccumulateGrad::apply` locks `mutex_`, `CopySlices::apply_impl` locks `mutex_`). The Python-facing code in `init.cpp` runs during module initialization (called from `THPAutograd_initFunctions` in `Module.cpp`), which is serialized by the import lock. The `static bool flag` patterns in `accumulate_grad.cpp:160` and `tensor.cpp:72` use C++ magic statics (guaranteed thread-safe initialization by the C++ standard). The `PyTuple_GET_ITEM` calls in the constructor functors (`DelayedErrorCtor`, `UndefinedGradCtor`) operate on the `args` tuple owned by the calling frame, which is safe. Static `PyTypeObject` variables and `PyGetSetDef` arrays are written once at init time and only read thereafter. The global `cpp_function_types_map` (defined in `python_cpp_function.cpp`, used by `registerCppFunction` called from `init.cpp`) is populated during module init and read at runtime, but that is outside the scope of these files.
