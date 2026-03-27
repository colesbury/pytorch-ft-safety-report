# Free-Threading Safety Audit: jit/backends/nnapi

**Overall Risk: Low-Medium**

## Files Reviewed
- `torch/csrc/jit/backends/nnapi/nnapi_backend_lib.cpp`
- `torch/csrc/jit/backends/nnapi/nnapi_backend_preprocess.cpp`

## Findings

### 1. NnapiBackend::execute() -- Lazy init of comp_ and out_templates_ (Medium)
**File:** `nnapi_backend_lib.cpp`, lines 49-53
```cpp
c10::impl::GenericList execute(
    c10::IValue handle,
    c10::impl::GenericList inputs) override {
  ...
  if (comp_ == nullptr) {
    init(handle, tensorInp);
  }
  TORCH_CHECK(comp_ != nullptr)
  ...
  comp_->run(fixed_inputs, outputs.vec());
  ...
}
```
The `NnapiBackend` instance has mutable members `comp_` (a `unique_ptr<NnapiCompilation>`) and `out_templates_` (a `List<Tensor>`), which are lazily initialized on the first call to `execute()`. This is a classic check-then-act race: if two threads call `execute()` on the same `NnapiBackend` instance concurrently, both could see `comp_ == nullptr`, both could call `init()`, and the writes to `comp_` and `out_templates_` would race.

In practice, each backend instance is typically owned by a single lowered module, and inference on the same module from multiple threads is not common. However, the pattern is inherently unsafe under free-threading.

**Severity: Medium** -- Lazy initialization without synchronization. If the same backend instance were used from multiple threads, this would be a data race on `comp_` and `out_templates_`.

**Recommendation:** Protect the lazy init with a `std::once_flag` / `std::call_once`, or initialize `comp_` during `compile()` if input shapes are known at that point.

### 2. NnapiBackend -- Per-instance state, no static mutable globals (Safe)
**File:** `nnapi_backend_lib.cpp`
All mutable state (`comp_`, `out_templates_`) is per-instance. No static mutable C++ state is introduced. The static `cls` registration is a safe static initializer.

**Severity:** None for static state.

### 3. nnapi_backend_preprocess.cpp -- py::gil_scoped_acquire (Low)
**File:** `nnapi_backend_preprocess.cpp`, line 32
```cpp
static c10::IValue preprocess(...) {
  py::gil_scoped_acquire gil;
  py::object pyModule = py::module_::import("torch.backends._nnapi.prepare");
  ...
}
```
This function explicitly acquires the GIL before calling into Python. Under free-threading (Python 3.14t/nogil), `py::gil_scoped_acquire` is a no-op since the GIL does not exist. However, the Python C API calls made through pybind11 (`py::module_::import`, attribute access, function calls) should still be safe under free-threading because pybind11 operations on Python objects use the appropriate critical sections internally (assuming a free-threading-aware pybind11).

The function holds no static mutable C++ state. All variables are local.

**Severity: Low** -- The `gil_scoped_acquire` becomes a no-op under free-threading, but the pybind11 Python interop should handle this. No C++-level races introduced.

### 4. Static registration `pre_reg` and `cls` (Safe)
**File:** `nnapi_backend_preprocess.cpp`, lines 112-114; `nnapi_backend_lib.cpp`, line 132
```cpp
static auto pre_reg =
    torch::jit::backend_preprocess_register(backend_name, preprocess);
static auto cls = torch::jit::backend<NnapiBackend>(backend_name);
```
Static initializer registrations. Run once during module load. Safe.

**Severity:** None

## Summary

The primary concern is the lazy initialization pattern in `NnapiBackend::execute()`, where `comp_` is initialized on first call without synchronization. If the same backend instance were accessed from multiple threads, this would be a data race. The preprocess function properly acquires the GIL for Python interop and holds no static mutable C++ state. All static registrations are safe.
