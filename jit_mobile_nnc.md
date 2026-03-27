# Free-Threading Safety Audit: jit_mobile_nnc

## Files Audited
- `jit/mobile/nnc/aot_compiler.cpp`
- `jit/mobile/nnc/aot_compiler.h`
- `jit/mobile/nnc/backend.cpp`
- `jit/mobile/nnc/context.cpp`
- `jit/mobile/nnc/context.h`
- `jit/mobile/nnc/registry.cpp`
- `jit/mobile/nnc/registry.h`

## Summary

This group implements the NNC (Neural Network Compiler) backend for mobile, including ahead-of-time compilation, a runtime context/compilation unit, and a kernel registry. Note that the NNC backend is currently disabled (the backend registration and preprocess registration are commented out). The code is pure C++ with no Python C API usage.

## Findings

### 1. NNC backend is disabled (informational)
- **File**: `jit/mobile/nnc/aot_compiler.cpp`, line 444; `jit/mobile/nnc/backend.cpp`, line 52
- **Severity**: None (informational)
- **Description**: Both the backend registration (`static const auto cls = torch::jit::backend<NNCBackend>("nnc")`) and the preprocess registration (`static auto reg = torch::jit::backend_preprocess_register("nnc", preprocess)`) are commented out with a TODO. The AOT compilation code inside `aot_compiler.cpp` is also entirely inside a block comment. This means the NNC mobile backend is not active and poses no runtime risk.

### 2. NNCKernelRegistry (registry.cpp/registry.h)
- **File**: `jit/mobile/nnc/registry.cpp`, `jit/mobile/nnc/registry.h`
- **Severity**: Low
- **Description**: `C10_DEFINE_REGISTRY(NNCKernelRegistry, NNCKernel)` defines a global registry for NNC kernels. The `c10::Registry` class uses internal synchronization (a mutex) for `Register` and `Create` operations. `has_nnc_kernel()` calls `Has()` and `get_nnc_kernel()` calls `Create()`, both of which are synchronized. Thread-safe.

### 3. `Function::init_execution_state()` lazy initialization (context.cpp)
- **File**: `jit/mobile/nnc/context.cpp`, lines 198-241
- **Severity**: Medium
- **Description**: `Function::init_execution_state()` checks `if (execution_state_ != nullptr) return;` and then lazily allocates the execution state. The `execution_state_` member is a `mutable std::unique_ptr<ExecutionState>`. If two threads call `run()` on the same `Function` concurrently, both could observe `execution_state_ == nullptr`, enter the initialization path, and race on assigning to `execution_state_`.
- **Current GIL Dependence**: If the NNC backend were active, it would be called from the JIT interpreter, which under GIL serializes Python thread execution.
- **Recommendation**: Since the NNC backend is currently disabled, this is a latent issue. If re-enabled, use `std::call_once` or `std::atomic` for the initialization check. Alternatively, initialize execution state eagerly during deserialization.

### 4. `Function::run()` mutates `execution_state_->arguments_` (context.cpp)
- **File**: `jit/mobile/nnc/context.cpp`, lines 243-297
- **Severity**: Medium (if NNC were enabled)
- **Description**: `Function::run()` fills in `args` (a reference to `execution_state_->arguments_`) with input/output pointers for each invocation. Since `execution_state_` is shared across calls (it is a member of `Function`), concurrent calls to `run()` on the same `Function` would race on the `arguments_` vector. This is a fundamental design issue: the execution state is per-Function, not per-invocation.
- **Current GIL Dependence**: Not applicable since NNC is disabled.
- **Recommendation**: If re-enabled, `ExecutionState` should be made per-invocation (allocated on the stack or in a thread-local pool) rather than cached as a member.

### 5. `CompilationUnit` functions map (context.cpp/context.h)
- **File**: `jit/mobile/nnc/context.cpp`, lines 299-341; `jit/mobile/nnc/context.h`, lines 193-221
- **Severity**: None
- **Description**: `CompilationUnit::functions_` is an `std::unordered_map` that is populated during construction (from deserialized IValues) and only read afterward via `find_function()` and `run()`. Since the map is populated once and then read-only, this is safe as long as the CompilationUnit is fully constructed before being shared across threads.

### 6. AOT compiler functions (aot_compiler.cpp) - all commented out
- **File**: `jit/mobile/nnc/aot_compiler.cpp`
- **Severity**: None
- **Description**: The entire AOT compiler implementation is inside a block comment. No active code to audit.

## No Python C API Usage

None of the files in this group use the Python C API.

## Overall Risk Assessment: LOW (because NNC backend is disabled)

The NNC backend is currently disabled, so the identified issues (lazy execution state initialization, shared mutable execution state in `Function::run()`) are latent rather than active. If the NNC backend is re-enabled, the `Function::run()` method would need a redesign to avoid sharing mutable execution state across concurrent invocations. The kernel registry uses proper synchronization and is safe.
