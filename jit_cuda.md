# Free-Threading Safety Audit: jit_cuda

## Files Reviewed
- `torch/csrc/jit/cuda/cuda.h`

## Overview

This single header file defines TorchBind wrappers (`torch::jit::CUDAStream` and `torch::jit::CUDAEvent`) around `c10::cuda::CUDAStream` and `at::cuda::CUDAEvent`, and registers them via `TORCH_LIBRARY(cuda, m)`. No Python C API is used directly.

## Issues

### Issue 1: `TORCH_LIBRARY(cuda, m)` registration

**Location:** `cuda.h`, lines 151-177

The `TORCH_LIBRARY` macro registers the `cuda::Stream` and `cuda::Event` classes with the TorchBind system. This registration executes at static initialization time.

**Risk:** None -- runs before user threads.

### Issue 2: CUDAStream and CUDAEvent wrapper classes

The wrapper classes hold a `unique_ptr` to the underlying `c10::cuda::CUDAStream` or `at::cuda::CUDAEvent`. Each instance owns its resource. The classes themselves have no shared mutable state.

Thread safety of individual instances depends on the underlying c10/ATen objects, but sharing a single `CUDAStream` or `CUDAEvent` wrapper across threads without external synchronization would be unsafe (same as the underlying CUDA objects). This is the expected contract.

**Risk:** None -- normal per-instance ownership semantics.

## Summary

No free-threading safety issues. This file contains only TorchBind wrapper classes with per-instance state and static-time registration. No Python C API, no global mutable state, no lazy initialization patterns.
