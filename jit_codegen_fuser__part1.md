# Free-Threading Safety Audit: jit_codegen_fuser__part1

## Files Reviewed
- `torch/csrc/jit/codegen/fuser/arg_spec.h`
- `torch/csrc/jit/codegen/fuser/codegen.cpp`
- `torch/csrc/jit/codegen/fuser/codegen.h`
- `torch/csrc/jit/codegen/fuser/compiler.cpp`
- `torch/csrc/jit/codegen/fuser/compiler.h`
- `torch/csrc/jit/codegen/fuser/executor.cpp`
- `torch/csrc/jit/codegen/fuser/executor.h`
- `torch/csrc/jit/codegen/fuser/fallback.cpp`
- `torch/csrc/jit/codegen/fuser/fallback.h`
- `torch/csrc/jit/codegen/fuser/fused_kernel.h`
- `torch/csrc/jit/codegen/fuser/interface.cpp`
- `torch/csrc/jit/codegen/fuser/interface.h`
- `torch/csrc/jit/codegen/fuser/kernel_cache.cpp`
- `torch/csrc/jit/codegen/fuser/kernel_cache.h`
- `torch/csrc/jit/codegen/fuser/kernel_spec.h`
- `torch/csrc/jit/codegen/fuser/partition_desc.h`
- `torch/csrc/jit/codegen/fuser/tensor_desc.h`
- `torch/csrc/jit/codegen/fuser/tensor_info.h`

## Overview

This group implements the legacy TorchScript CPU/CUDA fuser: kernel code generation, compilation, caching, and execution. The code is primarily C++ with no direct Python C API usage. It already contains significant internal locking (kernel cache, fusion backends map, per-KernelSpec mutex).

## Issues

### Issue 1: `debug_fusion` lazy init without synchronization

**Location:** `compiler.cpp`, lines 56-68

```cpp
static int debug_fusion{-1};

int debugFuser() {
  if (debug_fusion < 0) {
    const auto debug_env = c10::utils::get_env("PYTORCH_FUSION_DEBUG");
    debug_fusion = debug_env ? atoi(debug_env->c_str()) : 0;
  }
  return debug_fusion;
}
```

This is a classic lazy init race: multiple threads could read `-1`, both call `get_env`, and both write. Since the result is always the same value (determined by env var), the worst case is a benign redundant write. However, the non-atomic read-modify-write is technically a data race under C++ memory model.

**Risk:** Low -- the value is deterministic from the environment; worst case is redundant work, not wrong behavior.

**Recommended Fix:** Use `std::atomic<int>` or `std::call_once`.

### Issue 2: `cpu_fuser_enabled` and `gpu_fuser_enabled` are non-atomic globals

**Location:** `interface.cpp`, lines 13-21

```cpp
#ifdef TORCH_ENABLE_LLVM
bool cpu_fuser_enabled = true;
#else
static bool cpu_fuser_enabled = false;
#endif

static bool gpu_fuser_enabled = true;
```

These are read by `canFuseOnCPU()` / `canFuseOnGPU()` and written by `overrideCanFuseOnCPU()` / `overrideCanFuseOnGPU()`. Without the GIL, concurrent read/write is a data race.

**Risk:** Low -- typically set during initialization and read during execution; the values are booleans so tearing is not a concern on most architectures.

**Recommended Fix:** Change to `std::atomic<bool>`.

### Issue 3: `static std::unordered_map<NodeKind, RHSTemplate> simple_map_ops` in `encodeRHS`

**Location:** `codegen.cpp`, line 178

```cpp
static std::unordered_map<NodeKind, RHSTemplate> simple_map_ops = { ... };
```

This is a function-local static that is initialized once (C++11 guarantees thread-safe initialization of function-local statics). After initialization, it is only read, never mutated. No issue.

**Risk:** None.

### Issue 4: `static RegisterOperators reg_fused_operators` in fallback.cpp

**Location:** `fallback.cpp`, line 20

Static registration object constructed at file scope. This executes during static initialization, before any user threads exist.

**Risk:** None.

### Issue 5: `static auto dim_calc` template in codegen.cpp

**Location:** `codegen.cpp`, line 20

Read-only after initialization. Thread-safe.

**Risk:** None.

## Summary

The fuser code is mostly well-protected with internal mutexes (kernel cache, fusion backends). The two issues worth addressing are:

1. **`debug_fusion` lazy init** -- technically a data race, but benign. Fix with `std::atomic<int>`.
2. **`cpu_fuser_enabled` / `gpu_fuser_enabled`** -- non-atomic booleans read/written from multiple threads. Fix with `std::atomic<bool>`.

No Python C API is used in any of these files; all concerns are about C++ shared mutable state.
