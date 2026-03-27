# Free-Threading Safety Audit: jit/backends

**Overall Risk: Medium**

## Files Reviewed
- `torch/csrc/jit/backends/backend.h`
- `torch/csrc/jit/backends/backend_debug_handler.cpp`
- `torch/csrc/jit/backends/backend_debug_handler.h`
- `torch/csrc/jit/backends/backend_debug_info.cpp`
- `torch/csrc/jit/backends/backend_debug_info.h`
- `torch/csrc/jit/backends/backend_detail.cpp`
- `torch/csrc/jit/backends/backend_detail.h`
- `torch/csrc/jit/backends/backend_exception.h`
- `torch/csrc/jit/backends/backend_init.cpp`
- `torch/csrc/jit/backends/backend_init.h`
- `torch/csrc/jit/backends/backend_interface.cpp`
- `torch/csrc/jit/backends/backend_interface.h`
- `torch/csrc/jit/backends/backend_preprocess.h`
- `torch/csrc/jit/backends/backend_resolver.cpp`
- `torch/csrc/jit/backends/backend_resolver.h`

## Findings

### 1. backendPreprocessFunctions() -- Unprotected static mutable map (Medium)
**File:** `backend_detail.cpp`, lines 62-67
```cpp
std::unordered_map<std::string, BackendPreprocessFunction>&
backendPreprocessFunctions() {
  static std::unordered_map<std::string, BackendPreprocessFunction>
      preprocess_functions;
  return preprocess_functions;
}
```
This is a Meyer's singleton returning a mutable reference to a static `unordered_map`. The functions `hasBackendPreprocessFunction()`, `registerBackendPreprocessFunction()`, and `getBackendPreprocessFunction()` all access this map without any synchronization.

Registration happens via static initializers (`backend_preprocess_register` in `backend_preprocess.h`), which run during module loading and are single-threaded in that context. Reads happen later during `codegen_backend_module()`. In practice, the temporal separation between registration (startup) and read (usage) makes races unlikely. However, if registration is done dynamically (e.g., from Python `import` that triggers a static initializer on a different thread), a read-write race could occur.

**Severity: Medium** -- Static mutable state accessed without locking. The registration-then-read pattern is common and normally safe due to temporal ordering, but there is no formal guarantee.

**Recommendation:** Either document that registration must complete before any reads, or protect with a mutex. A `std::shared_mutex` would allow concurrent reads.

### 2. BackendDebugInfoRecorder::unique_debug_handle_ -- Atomic but non-atomic load+increment pattern (Low)
**File:** `backend_debug_handler.cpp`, lines 5-21
```cpp
std::atomic<DebugHandleType> BackendDebugInfoRecorder::unique_debug_handle_{0};

int64_t BackendDebugInfoRecorder::getNextDebugHandle(const Node* node) {
  ...
  DebugHandleType debug_handle = unique_debug_handle_;
  ...
  unique_debug_handle_++;
  return debug_handle;
}
```
The static atomic `unique_debug_handle_` is read into a local variable and then separately incremented. This is not an atomic read-modify-write (like `fetch_add`). If two threads call `getNextDebugHandle()` concurrently, both could read the same value before either increments, resulting in duplicate debug handles.

However, `BackendDebugInfoRecorder` instances are typically used within a single `codegen_backend_module()` call. The static atomic ensures uniqueness across different recorder instances, but the load-then-increment pattern is still technically racy.

**Severity: Low** -- The `handles_to_inlined_callstack_ptrs_` map is per-instance (not static), so the worst case is duplicate handle values across concurrent compilations. Backend compilation is not typically concurrent.

**Recommendation:** Replace the two-step load + increment with `unique_debug_handle_.fetch_add(1)` for correctness.

### 3. backend<T> constructor -- Static local `cls` registration (Low)
**File:** `backend.h`, lines 96-111
```cpp
backend(const std::string& name) : backend_name_(name) {
  static auto cls = torch::class_<TBackendInterface>(kBackendsNamespace, name)
                        .def(torch::init<>())
                        ._def_unboxed(...)
                        ...;
}
```
The `static auto cls` inside the template constructor is initialized exactly once via the C++ "magic statics" (C++11 thread-safe static local initialization). This is safe.

However, each template instantiation `backend<T>` is typically used as a static global (e.g., `static auto cls = torch::jit::backend<CoreMLBackend>("coreml")`), so the constructor runs during static initialization. Multiple backends in different translation units are initialized in unspecified order but each TU's static init is single-threaded within that TU. The `torch::class_<>` registration itself must be internally thread-safe for this to work in the presence of concurrent `dlopen`.

**Severity: Low** -- Relies on `torch::class_` registration being safe during static init, which it is in practice.

### 4. backend_debug_info.cpp -- Static auto `cls` registration (Safe)
**File:** `backend_debug_info.cpp`, lines 7-16
```cpp
static auto cls = torch::class_<PyTorchBackendDebugInfo>(
                      kBackendUtilsNamespace,
                      kBackendDebugInfoClass)
                      .def(torch::init<>());
```
This is a static initializer that registers a custom class. It runs once during static initialization. Thread-safe by construction (single-threaded static init context).

**Severity:** None

### 5. codegen_backend_module() -- Static const CodeTemplate objects (Safe)
**File:** `backend_detail.cpp`, lines 172, 217, 265, 275
```cpp
static const auto create_backend_ct = at::jit::CodeTemplate(R"(...)");
```
These are `static const` CodeTemplate objects. They are initialized once (thread-safe via magic statics) and never mutated. Safe.

**Severity:** None

### 6. initJitBackendBindings() -- pybind11 module registration (Safe)
**File:** `backend_init.cpp`, lines 132-188
This function is called once during module initialization (`initJitBackendBindings(module)`). It registers Python bindings using pybind11. Module init is single-threaded by Python convention (even under free-threading, module init is protected by the import lock).

The lambda captures use Python C API calls (`py::module::import`, `py::cast`, etc.) which are called within the context of already-held GIL (pybind11 methods). Under free-threading, pybind11's interaction with Python objects needs to be evaluated at the pybind11 level, but the individual binding definitions here do not introduce additional races.

**Severity:** None

### 7. backend_resolver.cpp -- loweredModuleResolver() allocates each call (Safe)
**File:** `backend_resolver.cpp`, lines 63-67
```cpp
std::shared_ptr<Resolver> loweredModuleResolver() {
  std::shared_ptr<Resolver> resolver =
      std::make_shared<LoweredModuleResolver>();
  return resolver;
}
```
This creates a new resolver each call. No shared mutable state. Safe.

**Severity:** None

### 8. BackendRuntimeException -- No shared state (Safe)
**File:** `backend_exception.h`
The `BackendRuntimeException` class holds a `std::vector<int64_t> debug_handles` member that is per-instance. No static or shared mutable state. Safe.

**Severity:** None

## Summary

The primary concern is the unprotected static `backendPreprocessFunctions()` map in `backend_detail.cpp`. The temporal separation between registration (static init) and reads (later usage) provides practical safety, but there is no formal synchronization. The atomic debug handle counter uses a non-atomic load+increment pattern that should be replaced with `fetch_add`. The rest of the code is either stateless, uses safe static initialization, or operates on per-instance data.
