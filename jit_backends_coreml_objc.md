# Free-Threading Safety Audit: jit/backends/coreml/objc

**Overall Risk: Low**

## Files Reviewed
- `torch/csrc/jit/backends/coreml/objc/PTMCoreMLCompiler.h`
- `torch/csrc/jit/backends/coreml/objc/PTMCoreMLExecutor.h`
- `torch/csrc/jit/backends/coreml/objc/PTMCoreMLFeatureProvider.h`
- `torch/csrc/jit/backends/coreml/objc/PTMCoreMLModelWrapper.h`
- `torch/csrc/jit/backends/coreml/objc/PTMCoreMLTensorSpec.h`

Note: These are all header files with interface declarations. The Objective-C implementations (`.mm` files) are not in this group. The audit is limited to what can be observed from the headers.

## Findings

### 1. PTMCoreMLExecutor::model -- `atomic` property (Safe)
**File:** `PTMCoreMLExecutor.h`, line 9
```objc
@property(atomic, strong) MLModel* model;
```
The `model` property is declared `atomic`, which provides thread-safe getter/setter at the Objective-C level. This is appropriate for a property that might be set from one thread and read from another.

**Severity:** None

### 2. MLModelWrapper -- Manual retain/release (Low)
**File:** `PTMCoreMLModelWrapper.h`, lines 10-36
```cpp
class MLModelWrapper : public CustomClassHolder {
 public:
  PTMCoreMLExecutor* executor;
  std::vector<TensorSpec> outputs;
  ...
  MLModelWrapper(PTMCoreMLExecutor* executor) : executor(executor) {
    [executor retain];
  }
  ~MLModelWrapper() {
    [executor release];
  }
};
```
The `MLModelWrapper` manually retains and releases an Objective-C object. The copy/move constructors also retain. This is correct for single-threaded use but the `executor` and `outputs` fields are public with no synchronization. If an `MLModelWrapper` instance were shared across threads, concurrent access to `executor` or `outputs` would race.

However, `MLModelWrapper` extends `CustomClassHolder` (reference-counted), and typical usage wraps it in an `intrusive_ptr`. The C++ object is typically accessed from the backend `execute()` method, which is not expected to be called concurrently on the same model handle.

**Severity: Low** -- No synchronization on public fields, but the usage pattern (single-threaded access per model handle) prevents races in practice.

### 3. PTMCoreMLTensorSpec -- scalar_type() helper (Safe)
**File:** `PTMCoreMLTensorSpec.h`, lines 13-24
```cpp
static inline c10::ScalarType scalar_type(const std::string& type_string) { ... }
```
This is a pure function with no mutable state. Safe.

**Severity:** None

### 4. PTMCoreMLCompiler, PTMCoreMLFeatureProvider -- Objective-C interfaces only (Not Auditable)
**Files:** `PTMCoreMLCompiler.h`, `PTMCoreMLFeatureProvider.h`
These are Objective-C interface declarations with class methods and instance methods. Without the implementations (`.mm` files), thread-safety cannot be fully assessed from the headers alone. The class methods on `PTMCoreMLCompiler` (e.g., `+setCacheDirectory:`, `+cacheDirectory`, `+compileModel:modelID:`, `+loadModel:backend:allowLowPrecision:error:`) could have shared mutable state in their implementations, but that is outside the scope of these headers.

**Severity:** Not auditable from headers alone.

## Summary

This group consists entirely of Objective-C header files and one C++ wrapper class. There are no Python C API concerns. The `MLModelWrapper` has public mutable fields without synchronization, but the usage pattern prevents concurrent access in practice. The Objective-C interfaces require inspection of their `.mm` implementations to fully audit, which are not included in this group.
