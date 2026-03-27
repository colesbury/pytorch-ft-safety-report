# Free-Threading Safety Audit: functorch

**Overall Risk: None**

## Files Reviewed
- `functorch/init.cpp`
- `functorch/init.h`

## Findings

No free-threading issues found.

## Summary

The `functorch` module defines pybind11 bindings for functional transforms (vmap, grad, jvp, functionalize). All state management operates on thread-local state (TLS dispatch key sets, dynamic layer stack) or on per-tensor state (BatchedTensorImpl, TensorWrapper, FunctionalTensorWrapper). The `initFuncTorchBindings` function is called once during module initialization. The helper functions like `_grad_increment_nesting`, `_vmap_increment_nesting`, etc., manipulate TLS state via `c10::AutogradState::get_tls_state()` and the dynamic layer stack, which are designed to be thread-local. No mutable static or global state is accessed from these Python-facing functions.
