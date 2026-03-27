# Free-Threading Safety Audit: jit_passes_quantization__part1

## Overall Risk: LOW

## Files Audited
- `jit/passes/quantization/dedup_module_uses.cpp`
- `jit/passes/quantization/dedup_module_uses.h`
- `jit/passes/quantization/finalize.cpp`
- `jit/passes/quantization/finalize.h`
- `jit/passes/quantization/fusion_passes.cpp`
- `jit/passes/quantization/fusion_passes.h`
- `jit/passes/quantization/helper.cpp`
- `jit/passes/quantization/helper.h`
- `jit/passes/quantization/insert_observers.cpp`
- `jit/passes/quantization/insert_observers.h`
- `jit/passes/quantization/insert_quant_dequant.cpp`
- `jit/passes/quantization/insert_quant_dequant.h`
- `jit/passes/quantization/quantization_patterns.h`
- `jit/passes/quantization/quantization_type.cpp`
- `jit/passes/quantization/quantization_type.h`
- `jit/passes/quantization/register_packed_params.cpp`
- `jit/passes/quantization/register_packed_params.h`

## Summary

The files contain many static data structures, but they are all effectively constant after initialization. The quantization passes operate on JIT modules and graphs passed as parameters, with no shared mutable global state.

## Detailed Analysis

### Issues Found

#### Issue 1: Static mutable vectors in helper.cpp (lines 21-166+) -- LOW

```cpp
static std::vector<std::string> _static_quantizable_call_funcs = {...};
static std::vector<std::string> _static_quantizable_aten_funcs = {...};
static std::vector<std::string> _dynamic_quantizable_call_funcs = {...};
// ... many more
```

These are file-scope `static` vectors initialized with string literals. While not declared `const`, they are populated at file-scope initialization and never modified. They serve as lookup tables. This is safe but should be `const` for clarity and correctness.

#### Issue 2: Static data in quantization_patterns.h (lines 285+) -- LOW

```cpp
static std::vector<QuantFusionInfo> quant_fusion_pattern_and_replacements() { ... }
static std::vector<QuantFusionInfo> dynamic_quant_fusion_pattern_and_replacements() { ... }
static std::vector<QuantFusionInfo> linear_prepack_unpack_patterns() { ... }
static std::vector<QuantFusionInfo> conv_prepack_unpack_patterns() { ... }
```

These are `static` functions (not static data) that return new vectors each call. Each translation unit gets its own copy. Safe.

#### Issue 3: Static data in helper.cpp (lines 179-212+) -- LOW

```cpp
static std::tuple<c10::QScheme, QParamVector> _per_tensor_asym_qparam = {...};
static std::tuple<c10::QScheme, QParamVector> _per_tensor_sym_qparam = {...};
static std::unordered_map<NodeKind, std::tuple<c10::QScheme, QParamVector>>
    _qparam_map = {...};
static AtenFuncArgs _observe_inputs_aten_func = {};
```

These are initialized at file scope with constant data and never modified. Safe, but should be `const`.

### No Issues

- **dedup_module_uses.cpp**: Local graph processing.
- **finalize.cpp**: Graph processing with local `RegisterPrePackingParams` function.
- **fusion_passes.cpp**: Local graph fusion operations.
- **insert_observers.cpp**: Complex but instance-local module processing.
- **insert_quant_dequant.cpp**: Local graph quantization operations.
- **quantization_type.cpp**: Enum-like type definitions.
- **register_packed_params.cpp**: Module attribute registration.

## Recommendations

1. **helper.cpp**: Mark the static lookup table vectors and maps as `const` to prevent accidental modification and make immutability explicit.
