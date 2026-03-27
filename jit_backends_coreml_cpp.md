# Free-Threading Safety Audit: jit/backends/coreml/cpp

**Overall Risk: Low**

## Files Reviewed
- `torch/csrc/jit/backends/coreml/cpp/backend.cpp`
- `torch/csrc/jit/backends/coreml/cpp/context.cpp`
- `torch/csrc/jit/backends/coreml/cpp/context.h`
- `torch/csrc/jit/backends/coreml/cpp/preprocess.cpp`

## Findings

### 1. g_coreml_ctx_registry -- Atomic pointer to mutable object (Low)
**File:** `context.cpp`, lines 7-18
```cpp
std::atomic<ContextInterface*> g_coreml_ctx_registry;

BackendRegistrar::BackendRegistrar(ContextInterface* ctx) {
  g_coreml_ctx_registry.store(ctx);
}

void setModelCacheDirectory(std::string path) {
  auto p = g_coreml_ctx_registry.load();
  if (p) {
    p->setModelCacheDirectory(std::move(path));
  }
}
```
The pointer itself is atomic, so the load/store of the pointer is safe. However, after loading the pointer, `setModelCacheDirectory()` calls a virtual method on the pointed-to `ContextInterface` object. If the object itself were replaced or destroyed between the load and the call, this would be a use-after-free. In practice, `BackendRegistrar` is used as a static global, registering the context once at startup, and the pointed-to object lives for the process lifetime, so this is safe.

Additionally, `setModelCacheDirectory()` on the `ContextInterface` implementation may not be thread-safe itself, but this is a configuration method typically called at setup time, not concurrently.

**Severity: Low** -- The atomic pointer provides safe publication. The lifetime model (register once at static init, read later) is sound. Concurrent calls to `setModelCacheDirectory` could race inside the implementation, but this is a setup-time API.

### 2. CoreMLBackend -- Server-side stub, always unavailable (Safe)
**File:** `backend.cpp`
```cpp
class CoreMLBackend : public torch::jit::PyTorchBackendInterface {
  ...
  bool is_available() override { return false; }
};
static auto cls = torch::jit::backend<CoreMLBackend>("coreml");
```
This is a stub backend that always returns `false` from `is_available()` and throws from `compile()` and `execute()`. It has no mutable state. The static registration is safe (static initializer).

**Severity:** None

### 3. preprocess() in preprocess.cpp -- Python C API usage (Low)
**File:** `preprocess.cpp`, lines 17-34
```cpp
c10::IValue preprocess(...) {
  py::object pyModule =
      py::module_::import("torch.backends._coreml.preprocess");
  py::object pyMethod = pyModule.attr("preprocess");
  py::dict modelDict =
      pyMethod(mod, torch::jit::toPyObject(method_compile_spec));
  ...
}
```
This function calls into Python via pybind11 (`py::module_::import`, attribute access, function calls). Under the GIL-based model, pybind11 calls are safe because the GIL serializes access. Under free-threading, pybind11 operations on Python objects must be individually evaluated for thread safety.

However, this `preprocess` function is called from `codegen_backend_module()` during backend lowering, which is a compilation-time operation typically invoked from a single thread. The function does not hold any static mutable state itself.

**Severity: Low** -- The Python interop is handled by pybind11. No C++ thread-safety issues introduced by this code.

### 4. Static registration `pre_reg` (Safe)
**File:** `preprocess.cpp`, line 36-37
```cpp
static auto pre_reg =
    torch::jit::backend_preprocess_register("coreml", preprocess);
```
Static initializer registration. Runs once during module load. Safe.

**Severity:** None

## Summary

This group is low risk. The CoreML C++ backend is a server-side stub with no real functionality (always unavailable). The context registry uses an atomic pointer with a register-once-read-later pattern. The preprocess function delegates to Python and holds no static mutable C++ state.
