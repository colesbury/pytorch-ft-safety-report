# Free-Threading Safety Audit: jit_tensorexpr__part4

## Files Audited
- `torch/csrc/jit/tensorexpr/ir_verifier.cpp`
- `torch/csrc/jit/tensorexpr/ir_verifier.h`
- `torch/csrc/jit/tensorexpr/kernel.cpp`
- `torch/csrc/jit/tensorexpr/kernel.h`
- `torch/csrc/jit/tensorexpr/llvm_codegen.cpp`
- `torch/csrc/jit/tensorexpr/llvm_codegen.h`
- `torch/csrc/jit/tensorexpr/llvm_jit.cpp`
- `torch/csrc/jit/tensorexpr/llvm_jit.h`
- `torch/csrc/jit/tensorexpr/loopnest.cpp`
- `torch/csrc/jit/tensorexpr/loopnest.h`
- `torch/csrc/jit/tensorexpr/loopnest_randomization.cpp`
- `torch/csrc/jit/tensorexpr/loopnest_randomization.h`
- `torch/csrc/jit/tensorexpr/lowerings.cpp`

## Summary

This group covers IR verification, the TensorExpr kernel (the main entry point for NNC fusion), LLVM code generation and JIT, loop nest transformations, loop nest randomization, and NNC lowerings registry. This is the most concerning group due to several mutable global configuration variables.

## Issues Found

### Issue 1: Multiple mutable static globals for TensorExpr configuration
**File:** `torch/csrc/jit/tensorexpr/kernel.cpp` (lines 41-48)
**Severity:** High
**Description:** Seven mutable static variables control TensorExpr behavior:
```cpp
static int te_cuda_pointwise_loop_levels = -1;
static int te_cuda_pointwise_block_count = -1;
static int te_cuda_pointwise_block_size = -1;
static bool fallback_allowed = false;
static bool te_generate_block_code = false;
static bool te_must_use_llvm_on_cpu = true;
static bool cat_wo_conditionals = true;
static bool opt_conditionals = false;
```
These are exposed via accessor functions that return mutable references (e.g., `getTECudaPointwiseLoopLevels()`, `getTEGenerateBlockCode()`, `getTEMustUseLLVMOnCPU()`, `getCatWoConditionals()`, `getOptConditionals()`). The `setFallbackAllowed()` function reads and writes `fallback_allowed` non-atomically.

Under free-threading, concurrent reads and writes to these variables are data races. Any thread calling `setFallbackAllowed(true)` while another thread calls `fallbackAllowed()` would race.

**Recommendation:** Convert to `std::atomic<int>` / `std::atomic<bool>`, or change the accessor functions to not return mutable references. For the int values, use atomic load/store with relaxed ordering (they are configuration knobs, not synchronization primitives).

### Issue 2: LLVM target configuration globals
**File:** `torch/csrc/jit/tensorexpr/llvm_codegen.cpp` (lines 90-104)
**Severity:** Medium
**Description:** Four mutable static globals exposed via reference-returning accessors:
```cpp
std::optional<std::string>& LLVMTargetTriple() {
  static std::optional<std::string> triple = std::nullopt;
  return triple;
}
std::optional<std::string>& LLVMTargetCPU() { ... }
std::optional<std::string>& LLVMTargetAttrs() { ... }
bool& LLVMAOTWorkflow() { ... }
```
These can be written from one thread and read from another during `LLVMCodeGenImpl` construction.

**Recommendation:** If these are configuration set once at startup, document that constraint. Otherwise, protect with a mutex or use atomic types.

### Issue 3: `llvmInitMutex` is correctly used
**File:** `torch/csrc/jit/tensorexpr/llvm_codegen.cpp` (lines 486, 529)
**Severity:** None
**Description:** A `static std::mutex llvmInitMutex` correctly protects LLVM initialization (specifically `TargetRegistry::lookupTarget` which is documented as not thread-safe). This is properly guarded with `std::lock_guard`.
**Status:** Safe. Already properly synchronized.

### Issue 4: NNC lowerings registry lazy initialization
**File:** `torch/csrc/jit/tensorexpr/lowerings.cpp` (lines 11-13, 1992)
**Severity:** Medium
**Description:** `getNNCLoweringRegistry()` returns a reference to a `static FunctionSchemaMap<NNCLoweringFunction>`. The lazy registration is triggered by:
```cpp
NNCLoweringFunction getStandardLoweringFor(const std::string& schema_str) {
  [[maybe_unused]] static const int once = nnc_lowerings_lazy_registration();
  ...
}
```
The `static const int once` ensures `nnc_lowerings_lazy_registration()` executes exactly once (magic statics). However, `RegisterNNCLoweringsFunction` constructors (called within `nnc_lowerings_lazy_registration`) insert into the registry. If `getNNCLoweringRegistry()` is also called from other threads or file-scope static `RegisterNNCLoweringsFunction` instances in other TUs, there could be a race between the lazy registration and other insertions.

In practice, `nnc_lowerings_lazy_registration()` runs all registrations in one shot within the magic static initializer, so the registry is fully populated before the first `find()` call returns. This is safe.
**Status:** Safe under the current usage pattern.

### Issue 5: Static `noSleef` set in llvm_codegen.cpp
**File:** `torch/csrc/jit/tensorexpr/llvm_codegen.cpp` (line 1964)
**Severity:** None
**Description:** `static std::unordered_set<std::string> noSleef` is initialized once (magic statics) and only read. Safe.
**Status:** Safe.

### Issue 6: Environment variable caching with `static const auto`
**File:** `torch/csrc/jit/tensorexpr/kernel.cpp` (lines 57, 69, 94)
**Severity:** None
**Description:** Several functions cache environment variable lookups in `static const auto enable_opt = c10::utils::get_env(...)`. These are initialized once (magic statics) and never modified. Safe.
**Status:** Safe.

## Concurrency Notes

- `ir_verifier.cpp`/`ir_verifier.h`: Pure visitor with instance-level state. Safe.
- `kernel.cpp`/`kernel.h`: The `TensorExprKernel` class itself uses instance-level state. The global config variables (Issue 1) are the concern.
- `llvm_jit.cpp`/`llvm_jit.h`: `PytorchLLVMJIT` wraps an LLVM ORC JIT. Instance-level state. Thread safety depends on LLVM's own JIT being safe for per-instance use.
- `loopnest.cpp`/`loopnest.h`: All methods are either instance methods or static methods operating on passed-in IR nodes. No global mutable state. The `static bool isValidIdentifierChar` at line 120 is a pure function. Safe.
- `loopnest_randomization.cpp`: Uses `std::mt19937` seeded per-call. No global mutable state. Safe.
- No Python C API calls in any of these files.
