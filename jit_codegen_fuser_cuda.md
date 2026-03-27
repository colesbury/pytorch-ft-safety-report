# Free-Threading Safety Audit: jit_codegen_fuser_cuda

## Files Reviewed
- `torch/csrc/jit/codegen/fuser/cuda/fused_kernel.cpp`
- `torch/csrc/jit/codegen/fuser/cuda/fused_kernel.h`
- `torch/csrc/jit/codegen/fuser/cuda/resource_strings.h`

## Overview

This group implements CUDA-side kernel compilation for the TorchScript fuser using NVRTC. It compiles CUDA kernels at runtime, loads them via the driver API, and launches them. No Python C API is used.

## Issues

### Issue 1: `static RegisterFusionBackend reg` at file scope

**Location:** `fused_kernel.cpp`, line 264

```cpp
RegisterFusionBackend reg(DeviceType::CUDA, createFusionKernel);
```

This calls `registerFusionBackend()` which acquires `fusionBackendLock()` (from `compiler.cpp`). It runs at static init time before user threads exist.

**Risk:** None.

### Issue 2: Resource string templates are all `static` / `constexpr` read-only data

**Location:** `resource_strings.h`, entire file

All strings are either `constexpr` or `static auto ... = at::jit::CodeTemplate(R"(...)")`. These are initialized once and never mutated.

**Risk:** None.

### Issue 3: `FusedKernelCUDA` construction and launch

**Location:** `fused_kernel.cpp`, lines 85-186, 192-238

The constructor performs NVRTC compilation and cuModuleLoad on a specific device. The `launch_raw` method uses `at::cuda::CUDAGuard` and CUDA driver API calls. These are not Python-facing and operate on per-kernel state.

The `launch_raw` method does save/restore `current_device` (lines 197-198, 237), which could be problematic if two threads share the same CUDA context, but this is a CUDA-level concern and is handled by the CUDA guard pattern used throughout PyTorch. The `FusedKernelCUDA` object itself is immutable after construction (all fields are set in the constructor and never modified afterward, except `maxBlocks_` which is also set in the constructor).

**Risk:** None for free-threading specifically. The CUDA device switching pattern is a pre-existing concern shared across all of PyTorch's CUDA code.

## Summary

No free-threading safety issues found in this group. All state is either const after initialization, or properly synchronized. No Python C API is used. The CUDA kernel compilation and launch paths use existing PyTorch CUDA guard patterns.
