# `kActivePyCompilerInterface` unprotected global `unique_ptr`

- **Status:** Open
- **Severity:** Significant
- **Tier:** Tier 2
- **Component:** compiled_autograd
- **Source report:** [dynamo_compiled_autograd_v2.md](../dynamo_compiled_autograd_v2.md)

- **Tier:** Tier 2 (requires concurrent compiled autograd use)
- **Shared state:** `kActivePyCompilerInterface` (file-static
  `unique_ptr<PyCompilerInterface>` in `compiled_autograd.cpp`, line 6).
- **Writer(s):** `PyCompilerGuard` constructor (line 16) moves a value in;
  destructor (line 21) resets to `nullptr`. Called from
  `_compiled_autograd_impl` during cache-miss compilation.
- **Reader(s):** `getPyCompilerInterface()` (line 8) returns a const reference
  to the `unique_ptr`. Called from `SwapSavedVariables::before(SavedVariable&)`
  (compiled_autograd.h line 819) during tracing, and from various autograd node
  implementations in libtorch_cpu (`custom_function.h:348`,
  `accumulate_grad.cpp:165,174`, `tensor.cpp:76,94,264`).
- **Race scenario:** Currently, all callers of `getPyCompilerInterface` are
  reached only during the cache-miss tracing phase, which runs under the
  compiled autograd mutex. The protection is therefore implicit. However,
  `getPyCompilerInterface()` is `TORCH_API` (public) and performs no
  synchronization. If a future code change calls it from outside the
  mutex-protected path, or if the mutex is relaxed for concurrent backwards,
  two threads could race on the `unique_ptr` (one writing via
  `PyCompilerGuard`, another reading via `getPyCompilerInterface`). This would
  corrupt the `unique_ptr` internals.
- **Suggested fix:** Make `kActivePyCompilerInterface` thread-local (it is
  semantically per-compilation-session), or protect it with a dedicated mutex
  or the compiled autograd mutex.
