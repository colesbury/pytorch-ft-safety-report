# Inductor Free-Threading Issues

Issues in `torch/csrc/inductor/` affecting free-threaded Python 3.14t.

## Architecture Summary

The inductor C++ runtime provides ahead-of-time (AOT) compiled model execution
infrastructure. Most of this code is pure C++ with its own locking (or lack
thereof). The GIL removal doesn't create new race conditions per se, but it
makes them far more likely to trigger: Python threads now run truly
concurrently, so the window for concurrent calls into these C++ APIs is wide
open.

Key subsystems:

- **AOTI Runtime** (`aoti_runtime/`): `AOTInductorModelContainer` manages a pool
  of `AOTInductorModel` instances with double-buffered constants. Has partial
  locking via `model_exec_mutex_` (shared_mutex), but management APIs bypass it.

- **AOTI Eager** (`aoti_eager/`): `AOTIPythonKernelHolder` is a dispatcher
  kernel functor — one instance per (op, dispatch key) pair, shared across all
  threads. Has **zero synchronization** on its kernel cache.

- **AOTI Runner/Package** (`aoti_runner/`, `aoti_package/`): Model loading,
  package extraction, pybind wrappers.

- **AOTI Torch Shim** (`aoti_torch/`): Thin C wrappers around ATen ops.
  Essentially stateless — very clean.

- **Static Launcher / cpp_prefix** (`static_launcher/`, `cpp_prefix.h`):
  Code included in all generated kernels, plus CUDA/XPU kernel launch helpers.

## Issues

| Severity | Component | Issue |
|----------|-----------|-------|
| SEVERE | aoti_eager | [`aoti_kernel_cache_` has zero synchronization](aoti-kernel-cache-no-synchronization.md) |
| SEVERE | aoti_runtime | [Model container management APIs bypass `model_exec_mutex_`](model-container-management-apis-bypass-mutex.md) |
| SEVERE | aoti_runtime | [`run_const_fold` inactive buffer path races with model state](run-const-fold-inactive-buffer-race.md) |
| SEVERE | aoti_package | [`load_json_file` static JSON object](load-json-file-static-object.md) |
| SEVERE | aoti_runner | [`getAOTIModelRunnerRegistry()` unsynchronized global map](aoti-model-runner-registry-unsynchronized.md) |
| Significant | aoti_runtime | [`use_secondary_` read without synchronization](use-secondary-read-without-synchronization.md) |
| Significant | aoti_runtime | [`constant_folded_` double-checked locking without atomics](constant-folded-double-checked-locking.md) |
| Significant | static_launcher | [TOCTOU on lazy kernel loading in generated code](lazy-kernel-loading-toctou.md) |
| Significant | cpp_wrapper | [TOCTOU on `loadLazyCompileFuncs()` statics](load-lazy-compile-funcs-toctou.md) |
| Significant | aoti_eager | [TOCTOU between `cache_lookup` and `cache_miss`](cache-lookup-cache-miss-toctou.md) |
| Minor | aoti_runtime | `get_constant_blob_ptr` lazy allocation not thread-safe (check-then-allocate, no lock) |
| Minor | aoti_runtime | `get_constants_map`/`get_constants_array` lazy allocation of secondary (same pattern) |
| Minor | aoti_runtime | `KernelContextGuard` move ctor leaves dangling TLS pointer (`kernel_context_tls.h`) |
| Minor | aoti_runtime | `run_const_fold` exception path leaves model with wrong constants (no RAII restore) |
| Minor | aoti_torch | `OSSProxyExecutor::call_function` uses non-const `operator[]` on shared map (`oss_proxy_executor.cpp:828`) |
| Minor | aoti_package | Windows `create_temp_dir` returns shared temp directory (no unique subdir) |

## Not Reported (safe or stateless)

- **AOTI Torch shim layer** (`aoti_torch/shim_common.cpp`, `shim_cpu.cpp`,
  `shim_cuda.cpp`, `shim_xpu.cpp`, `shim_mps.cpp`, `tensor_converter.cpp`,
  `utils.h`): ~4k lines of pure stateless wrappers around ATen ops. No shared
  mutable state. Each function converts a handle to a pointer, calls ATen,
  returns the result.

- **`cpp_prefix.h`**: Uses `thread_local` for caches, stateless helper
  functions. Clean.

- **AOTI runtime thread-local storage** (`thread_local.h`,
  `kernel_context_tls.h`): TLS variables are properly isolated. Only the
  `KernelContextGuard` move ctor issue noted above.

- **AOTI runtime utilities** (`utils.h`, `utils_cuda.h`, `utils_xpu.h`,
  `device_utils.h`, `arrayref_tensor.h`, `scalar_to_tensor.h`): Stateless
  utility/wrapper code.

- **`inductor_ops.cpp`**, **`resize_storage_bytes.cpp`**: Stateless operator
  implementations.
