# Free-Threading Safety Audit: jit/backends/xnnpack

**Overall Risk: Low**

## Files Reviewed
- `torch/csrc/jit/backends/xnnpack/xnnpack_backend_lib.cpp`
- `torch/csrc/jit/backends/xnnpack/xnnpack_backend_preprocess.cpp`
- `torch/csrc/jit/backends/xnnpack/xnnpack_graph_builder.cpp`
- `torch/csrc/jit/backends/xnnpack/xnnpack_graph_builder.h`

## Findings

### 1. XNNPackBackend -- Stateless backend class (Safe)
**File:** `xnnpack_backend_lib.cpp`, lines 22-105
The `XNNPackBackend` class has no member variables. Each call to `compile()` creates a new `XNNModelWrapper` wrapping an `XNNExecutor`. Each call to `execute()` operates on the executor obtained from the handle dictionary. There is no shared mutable state in the backend class itself.

**Severity:** None

### 2. XNNModelWrapper -- Per-instance state, no copy (Safe)
**File:** `xnnpack_backend_lib.cpp`, lines 12-20
```cpp
class XNNModelWrapper : public CustomClassHolder {
 public:
  XNNExecutor executor_;
  XNNModelWrapper(XNNExecutor executor) : executor_(std::move(executor)) {}
  XNNModelWrapper(const XNNModelWrapper& oldObject) = delete;
};
```
The wrapper is non-copyable and holds a single `XNNExecutor`. It is accessed through `intrusive_ptr` (via capsule). The public `executor_` field has no synchronization, but concurrent calls to `execute()` on the same model handle would need to be evaluated at a higher level. The `XNNExecutor::set_inputs()` method mutates `externals_`, so concurrent `execute()` calls on the same handle would race. However, this is a general backend contract issue (concurrent execute on the same handle is not expected).

**Severity: Low** -- Same-handle concurrent execution would race, but this is outside the expected usage pattern.

### 3. XNNGraph -- Per-instance build-time state (Safe)
**File:** `xnnpack_graph_builder.h`, `xnnpack_graph_builder.cpp`
The `XNNGraph` class holds per-instance state (`_serializer`, `_subgraph_ptr`, `_intermediate_tensors`, `_val_to_ids`, `_inputs`, `_outputs`). It is constructed, used to build and serialize a graph, and then destroyed. There is no static mutable state. All methods operate on instance members.

The `xnn_initialize()` call in the constructor and `xnn_deinitialize()` in the destructor are XNNPACK library-level calls. The XNNPACK documentation states `xnn_initialize` can be called multiple times safely, but `xnn_deinitialize` should only be called when all XNNPACK objects are destroyed. Calling these from multiple `XNNGraph` instances concurrently could be problematic at the XNNPACK library level, but this is an XNNPACK concern, not a PyTorch free-threading concern.

**Severity:** None for PyTorch-level thread safety.

### 4. preprocess() in xnnpack_backend_preprocess.cpp -- No shared state (Safe)
**File:** `xnnpack_backend_preprocess.cpp`, lines 28-122
The `preprocess` function creates local `XNNGraph` and other local variables. No static mutable state. The function is pure in terms of C++ side effects (all state is local or passed by argument).

**Severity:** None

### 5. Static registrations (Safe)
**File:** `xnnpack_backend_lib.cpp`, line 109; `xnnpack_backend_preprocess.cpp`, line 124
```cpp
static auto cls = torch::jit::backend<XNNPackBackend>(backend_name);
static auto pre_reg = backend_preprocess_register(backend_name, preprocess);
```
Static initializer registrations. Run once during module load. Safe.

**Severity:** None

## Summary

This group is low risk. The `XNNPackBackend` class is stateless. All mutable state is per-instance (`XNNModelWrapper`, `XNNGraph`) and not shared across threads. The preprocess function holds no static mutable C++ state. The only theoretical concern is concurrent execution on the same model handle (which would race on `XNNExecutor::externals_`), but this is outside the expected usage pattern.
