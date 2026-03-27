# Free-Threading Safety Audit: jit_codegen_fuser_cpu

## Files Reviewed
- `torch/csrc/jit/codegen/fuser/cpu/fused_kernel.cpp`
- `torch/csrc/jit/codegen/fuser/cpu/fused_kernel.h`
- `torch/csrc/jit/codegen/fuser/cpu/resource_strings.h`
- `torch/csrc/jit/codegen/fuser/cpu/temp_file.h`

## Overview

This group implements CPU-side kernel compilation for the TorchScript fuser. It compiles C++ code at runtime using the system compiler (g++/clang++/cl), loads the resulting shared library, and invokes the kernel. No Python C API is used.

## Issues

### Issue 1: `CompilerConfig` singleton with mutable `openmp` field

**Location:** `fused_kernel.cpp`, lines 211-213 and 279-283

```cpp
static CompilerConfig& getConfig() {
  static CompilerConfig config;
  return config;
}
```

In `runCompiler()`:
```cpp
if (config.openmp && r != 0) {
    config.openmp = false; // disable for future compiles
    return runCompiler(cpp_file, so_file);
}
```

The `CompilerConfig::openmp` field is read and conditionally set to `false` during compilation. If two threads attempt CPU kernel compilation concurrently and one fails with OpenMP, both could race on `config.openmp`. The read-then-write pattern is not atomic.

**Risk:** Low -- the worst case is both threads see `openmp == true`, both fail, both set `openmp = false`, and both retry. The final state is still correct. However, it is technically undefined behavior.

**Recommended Fix:** Make `openmp` an `std::atomic<bool>`.

### Issue 2: `env_list` global mutable state (Windows only)

**Location:** `fused_kernel.cpp`, line 33

```cpp
static std::vector<std::wstring> env_list;
```

On Windows, `activate()` populates `env_list` and `run()` reads it. Both are called from the `CompilerConfig` constructor and from `runCompiler`. The `CompilerConfig` constructor runs once (Meyers' singleton), but `run()` is called from `runCompiler()` which can be called concurrently. Since `env_list` is only written during `activate()` which runs during the singleton construction, and only read afterward, this is safe in practice.

**Risk:** None -- writes happen during singleton init, reads happen after.

### Issue 3: Static template strings

**Location:** `fused_kernel.cpp`, lines 37-42, 247-259

```cpp
static const std::string so_template = "/tmp/pytorch_fuserXXXXXX.so";
static const std::string cpp_template = "/tmp/pytorch_fuserXXXXXX.cpp";
static const std::string compile_string = ...;
static const std::string disas_string = ...;
```

All const after initialization. Thread-safe.

**Risk:** None.

### Issue 4: `static RegisterFusionBackend reg` at file scope

**Location:** `fused_kernel.cpp`, line 355

Registration happens at static init time before user threads.

**Risk:** None.

## Summary

The only actionable issue is the non-atomic read-modify-write of `CompilerConfig::openmp` in `runCompiler()`. The race is benign in practice (both paths converge to the same state), but should be fixed with `std::atomic<bool>` for correctness under the C++ memory model. No Python C API usage is present.
