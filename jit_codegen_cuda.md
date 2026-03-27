# Free-Threading Safety Audit: jit_codegen_cuda

## Files Reviewed
- `torch/csrc/jit/codegen/cuda/interface.cpp`
- `torch/csrc/jit/codegen/cuda/interface.h`

## Overview

This group contains the deprecated CUDA fuser (nvfuser) interface. All user-facing functions emit `TORCH_WARN_ONCE` and return false or no-op. The actual fuser functionality has been removed; the code is a thin stub.

## Issues

### Issue 1: Static `CudaFuserInterface` singleton via Meyers' singleton (Low Risk)

**Location:** `interface.cpp`, line 62-65

```cpp
CudaFuserInterface* getFuserInterface() {
  static CudaFuserInterface fuser_interface_;
  return &fuser_interface_;
}
```

The `CudaFuserInterface` struct holds raw function pointers that are meant to be set during registration (from a now-removed registration file). The Meyers' singleton initialization itself is thread-safe in C++11+, but the function pointer fields are subsequently read and potentially written without synchronization. In practice, registration happens during static initialization before threads are spawned, and the entire API is deprecated/non-functional, so the risk is negligible.

**Risk:** Low -- registration occurs at static init time; the API is deprecated.

### Issue 2: `cuda_fusion_guard_mode` is already atomic (No Issue)

**Location:** `interface.cpp`, line 5

```cpp
static std::atomic<bool> cuda_fusion_guard_mode{true};
```

Already uses `std::atomic<bool>`, so no data race.

## Summary

No actionable free-threading issues. The entire CUDA fuser interface is deprecated and effectively dead code. The singleton pattern for `CudaFuserInterface` uses Meyers' singleton (thread-safe construction) and is populated at static init time. No Python C API usage is present in these files.
