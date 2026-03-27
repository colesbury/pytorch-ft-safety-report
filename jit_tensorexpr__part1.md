# Free-Threading Safety Audit: jit_tensorexpr__part1

## Files Audited
- `torch/csrc/jit/tensorexpr/analysis.h`
- `torch/csrc/jit/tensorexpr/block_codegen.cpp`
- `torch/csrc/jit/tensorexpr/block_codegen.h`
- `torch/csrc/jit/tensorexpr/bounds_inference.cpp`
- `torch/csrc/jit/tensorexpr/bounds_inference.h`
- `torch/csrc/jit/tensorexpr/bounds_overlap.cpp`
- `torch/csrc/jit/tensorexpr/bounds_overlap.h`
- `torch/csrc/jit/tensorexpr/codegen.cpp`
- `torch/csrc/jit/tensorexpr/codegen.h`
- `torch/csrc/jit/tensorexpr/cpp_codegen.cpp`
- `torch/csrc/jit/tensorexpr/cpp_codegen.h`
- `torch/csrc/jit/tensorexpr/cpp_intrinsics.h`
- `torch/csrc/jit/tensorexpr/cuda_codegen.cpp`
- `torch/csrc/jit/tensorexpr/cuda_codegen.h`
- `torch/csrc/jit/tensorexpr/cuda_random.h`

## Summary

This group covers the TensorExpr IR analysis utilities, code generation infrastructure (base, C++, Block, CUDA), bounds inference, and bounds overlap analysis. These are largely computational C++ files with no Python C API usage. The main thread-safety concerns are global registries for code generators.

## Issues Found

### Issue 1: `RegisterCodeGenList` singleton with unprotected mutable map
**File:** `torch/csrc/jit/tensorexpr/codegen.cpp` (lines 34-64), `codegen.h` (lines 223-246)
**Severity:** Medium
**Description:** `RegisterCodeGenList::GetInstance()` returns a singleton (`static RegisterCodeGenList codegen_list`). The instance contains `std::unordered_map<std::string, StmtFactoryMethod> stmt_factory_methods_` which is mutated by `AddStmtFactoryMethod` (called from `RegisterCodeGen<T>` constructors at static initialization time) and read by `FindStmtFactoryMethod` (called at runtime when creating code generators).

If all registrations happen during static initialization (before `main()`), this is safe because static initializers within a single translation unit are sequential, and cross-TU ordering is the only concern. However, if any registration happens after threads start, or if `FindStmtFactoryMethod` is called during static initialization from another TU simultaneously, there is a data race.
```cpp
RegisterCodeGenList& RegisterCodeGenList::GetInstance() {
  static RegisterCodeGenList codegen_list;
  return codegen_list;
}
```
**Recommendation:** This is a standard startup-time registration pattern. Document that all `RegisterCodeGen<T>` instances must be created at static init time. Alternatively, add a read lock for `FindStmtFactoryMethod`.

### Issue 2: Static `RegisterCodeGen` instances at file scope
**Files:** `block_codegen.cpp` (line 361), `cpp_codegen.cpp` (line 403)
**Severity:** Low
**Description:** `static RegisterCodeGen<BlockCodeGen> block_codegen_reg("block_codegen")` and similar for CppCodeGen. These register into the global `RegisterCodeGenList` singleton. Since they are file-scope static objects, registration happens during static initialization. Safe as long as no queries happen concurrently with static initialization.
**Status:** Safe under normal startup conditions.

### Issue 3: Mutable static counter in `BlockCodeGen::GetUniqueFuncName`
**File:** `torch/csrc/jit/tensorexpr/block_codegen.cpp` (line 316)
**Severity:** Medium
**Description:** A `static int64_t counter = 0` is incremented non-atomically each time `GetUniqueFuncName` is called. Under free-threading, concurrent calls would race on this counter, potentially producing duplicate names.
```cpp
std::string BlockCodeGen::GetUniqueFuncName(const std::string& func_prefix) {
  static int64_t counter = 0;
  ++counter;
  int64_t value = counter;
  return func_prefix + "_" + std::to_string(value);
}
```
**Recommendation:** Change to `static std::atomic<int64_t> counter{0}` and use `counter.fetch_add(1)`.

### Issue 4: `ExtCallMemoryReuse::extCallFuncNameMap_` is static const
**File:** `torch/csrc/jit/tensorexpr/codegen.h` (line 119), `codegen.cpp` (line 265-266)
**Severity:** None
**Description:** This is a `static const std::unordered_map<std::string, std::string>` initialized from `makeExtCallFuncNameMap()`. Immutable after initialization. Safe.
**Status:** Safe.

## Concurrency Notes

- `analysis.h` contains only static member functions on template classes that operate on passed-in arguments. No global state. Safe.
- `bounds_inference.cpp` and `bounds_overlap.cpp` contain only file-scope static helper functions with no mutable state. Safe.
- `cuda_codegen.cpp` uses an instance-level `std::mutex eval_lock_` for protecting CUDA kernel compilation and execution. This is already properly synchronized.
- `cuda_random.h` contains only `static const` values. Safe.
- `cpp_intrinsics.h` contains only string constants. Safe.
- No Python C API calls in any of these files.
