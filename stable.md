# Free-Threading Safety Audit: stable

**Overall Risk: None**

## Files Reviewed
- `stable/accelerator.h`
- `stable/device.h`
- `stable/device_inl.h`
- `stable/device_struct.h`
- `stable/library.h`
- `stable/macros.h`
- `stable/ops.h`
- `stable/stableivalue_conversions.h`
- `stable/tensor.h`
- `stable/tensor_inl.h`
- `stable/tensor_struct.h`
- `stable/version.h`

## Findings

No free-threading issues found.

## Summary

The `stable` directory contains header-only definitions for the PyTorch stable ABI. These files define:

- **Value types** (`Device`, `Tensor`, `DeviceGuard`, `Stream`): These are self-contained objects that operate on their own state via C shim functions. No global mutable state.
- **Type conversion utilities** (`stableivalue_conversions.h`): Template specializations for converting between `StableIValue` and C++ types. All conversions are stateless and operate on function arguments.
- **Library registration** (`library.h`): `StableLibrary` and `StableTorchLibraryInit` classes for registering operator implementations. These are used at static init time via macros.
- **Constants** (`version.h`, `macros.h`): Compile-time constants and macros only.

None of these files contain mutable static/global state, Python C API calls, or any patterns that would be affected by free-threading. They are pure C++ headers that interact with PyTorch through stable C shim functions.
