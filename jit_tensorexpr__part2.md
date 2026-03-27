# Free-Threading Safety Audit: jit_tensorexpr__part2

## Files Audited
- `torch/csrc/jit/tensorexpr/eval.cpp`
- `torch/csrc/jit/tensorexpr/eval.h`
- `torch/csrc/jit/tensorexpr/exceptions.h`
- `torch/csrc/jit/tensorexpr/expr.cpp`
- `torch/csrc/jit/tensorexpr/expr.h`
- `torch/csrc/jit/tensorexpr/external_functions.cpp`
- `torch/csrc/jit/tensorexpr/external_functions.h`
- `torch/csrc/jit/tensorexpr/external_functions_codegen.cpp`
- `torch/csrc/jit/tensorexpr/external_functions_core.cpp`
- `torch/csrc/jit/tensorexpr/external_functions_core.h`
- `torch/csrc/jit/tensorexpr/external_functions_registry.cpp`
- `torch/csrc/jit/tensorexpr/external_functions_registry.h`
- `torch/csrc/jit/tensorexpr/fwd_decls.h`
- `torch/csrc/jit/tensorexpr/graph_opt.cpp`
- `torch/csrc/jit/tensorexpr/graph_opt.h`

## Summary

This group covers the TensorExpr IR evaluator, expression types, external function bridge (for calling ATen ops from NNC-generated code), external function registry, and graph optimization passes. The primary thread-safety concern is the global external function registry.

## Issues Found

### Issue 1: Global NNC external function registry without synchronization
**File:** `torch/csrc/jit/tensorexpr/external_functions_registry.cpp` (lines 5-8), `external_functions_registry.h` (lines 44-55)
**Severity:** Medium
**Description:** `getNNCFunctionRegistry()` returns a reference to a `static std::unordered_map<std::string, NNCExternalFunction>`. This map is written to by `RegisterNNCExternalFunction` constructors (which are file-scope static objects) and read at runtime when executing external calls from generated code.

Registration happens from many files (`external_functions.cpp` has ~40 registrations, `external_functions_codegen.cpp` has ~100+). All are file-scope statics, so registration occurs during static initialization.
```cpp
std::unordered_map<std::string, NNCExternalFunction>& getNNCFunctionRegistry() {
  static std::unordered_map<std::string, NNCExternalFunction> func_registry_;
  return func_registry_;
}
```
**Recommendation:** This follows the standard startup-time registration pattern. Safe as long as all `RegisterNNCExternalFunction` instances are file-scope statics created before `main()`. Consider adding a read-only accessor for runtime lookups to make the contract clear.

### Issue 2: `RegisterCodeGen<SimpleIREvaluator>` at file scope
**File:** `torch/csrc/jit/tensorexpr/eval.cpp` (line 13)
**Severity:** Low
**Description:** `static RegisterCodeGen<SimpleIREvaluator> ir_eval_codegen_reg("simple_ir_eval")` registers into the global codegen list during static initialization. Same pattern as Issue 1 in part1.
**Status:** Safe under normal startup conditions (see part1 Issue 1).

### Issue 3: Static `RegisterNNCExternalFunction` instances in external_functions.cpp and external_functions_codegen.cpp
**Files:** `external_functions.cpp` (lines 1447-1563), `external_functions_codegen.cpp` (lines 2886-3290)
**Severity:** Low
**Description:** Many `const static RegisterNNCExternalFunction` instances at file scope. All register into the global registry during static initialization. Since the map itself is lazily initialized via magic statics, the first access triggers safe initialization. Subsequent registrations from the same or different TUs during static init are ordered within a TU but not across TUs -- however, the map insertions themselves are safe because static initialization is single-threaded in practice (before `main()`).
**Status:** Safe under normal conditions.

## Concurrency Notes

- `eval.cpp` and `eval.h`: The `SimpleIREvaluator` class uses instance-level state. Thread-safe when instances are not shared. The evaluator stores intermediate computation results in per-instance vectors.
- `expr.cpp` and `expr.h`: Define expression node types. All `static` methods are factory methods that create new nodes. No global mutable state. Safe.
- `exceptions.h`: Only defines exception types. Safe.
- `external_functions.cpp`: The actual bridge functions (e.g., `nnc_aten_conv2d`) are stateless -- they receive all inputs via parameters and call ATen ops. Safe for concurrent invocation.
- `external_functions_core.cpp` and `external_functions_core.h`: Utility functions for tensor construction from raw pointers. Stateless. Safe.
- `graph_opt.cpp`: All static functions are pure graph transformation helpers. No global mutable state. Safe.
- `fwd_decls.h`: Only forward declarations. Safe.
- No Python C API calls in any of these files.
