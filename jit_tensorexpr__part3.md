# Free-Threading Safety Audit: jit_tensorexpr__part3

## Files Audited
- `torch/csrc/jit/tensorexpr/half_support.h`
- `torch/csrc/jit/tensorexpr/hash_provider.cpp`
- `torch/csrc/jit/tensorexpr/hash_provider.h`
- `torch/csrc/jit/tensorexpr/intrinsic_symbols.cpp`
- `torch/csrc/jit/tensorexpr/intrinsic_symbols.h`
- `torch/csrc/jit/tensorexpr/ir.cpp`
- `torch/csrc/jit/tensorexpr/ir.h`
- `torch/csrc/jit/tensorexpr/ir_cloner.cpp`
- `torch/csrc/jit/tensorexpr/ir_cloner.h`
- `torch/csrc/jit/tensorexpr/ir_mutator.cpp`
- `torch/csrc/jit/tensorexpr/ir_mutator.h`
- `torch/csrc/jit/tensorexpr/ir_printer.cpp`
- `torch/csrc/jit/tensorexpr/ir_printer.h`
- `torch/csrc/jit/tensorexpr/ir_simplifier.cpp`
- `torch/csrc/jit/tensorexpr/ir_simplifier.h`

## Summary

This group covers the TensorExpr IR core: node types, IR visitor/mutator/cloner infrastructure, printing, hash computation, simplification, half-precision support, and intrinsic symbols. These are pure C++ compiler infrastructure files with no Python C API usage. Almost all code is stateless or uses instance-level state.

## Issues Found

### Issue 1: Static `SymbolAddress` array in intrinsic_symbols.cpp
**File:** `torch/csrc/jit/tensorexpr/intrinsic_symbols.cpp` (line 133)
**Severity:** None
**Description:** `static SymbolAddress symbolAddresses[]` is initialized with function pointer addresses derived from standard math library functions. This is effectively a constant array (the function pointers don't change after initialization), and `getIntrinsicSymbols()` returns a `c10::ArrayRef` over it. Thread-safe for concurrent reads.
**Status:** Safe.

### Issue 2: Static `std::locale` in ir_printer.cpp
**File:** `torch/csrc/jit/tensorexpr/ir_printer.cpp` (line 51)
**Severity:** None
**Description:** `static std::locale c_locale("C")` is initialized once via magic statics. `std::locale` objects are immutable once constructed (they use reference counting internally which is thread-safe in modern standard library implementations). Using `imbue` with this locale on a per-instance stream is safe.
**Status:** Safe.

## Concurrency Notes

- `half_support.h`: Contains `HalfChecker` and `HalfRewriter` IR mutators. All instance-level state. Static methods (`isHalf`) are pure functions. Safe.
- `hash_provider.cpp`/`hash_provider.h`: `HashProvider` uses per-instance caches (`exprToHash_`, `stmtToHash_`). Safe when instances are not shared.
- `ir.cpp`/`ir.h`: Define IR node types with factory methods. No global mutable state. All `static` methods are pure constructors. Safe.
- `ir_cloner.cpp`/`ir_cloner.h`: `IRCloner` is a visitor that produces a deep copy. Instance-level state only. Safe.
- `ir_mutator.cpp`/`ir_mutator.h`: `IRMutator` base class. Instance-level state only. Safe.
- `ir_printer.cpp`/`ir_printer.h`: `IRPrinter` accumulates output into a per-instance ostream. Safe when instances are not shared.
- `ir_simplifier.cpp`/`ir_simplifier.h`: Contains `PolynomialTransformer`, `TermExpander`, `IRSimplifier`. All use instance-level state or are stateless static methods. The helper functions at file scope are all pure functions. Safe.
- No Python C API calls in any of these files.
