# Free-Threading Audit: autograd__part2

**Files:**
- `torch/csrc/autograd/custom_function.cpp`
- `torch/csrc/autograd/custom_function.h`
- `torch/csrc/autograd/edge.h`
- `torch/csrc/autograd/engine.cpp`
- `torch/csrc/autograd/engine.h`
- `torch/csrc/autograd/forward_grad.cpp`
- `torch/csrc/autograd/forward_grad.h`
- `torch/csrc/autograd/function.cpp`
- `torch/csrc/autograd/function.h`
- `torch/csrc/autograd/function_hook.h`
- `torch/csrc/autograd/grad_mode.h`
- `torch/csrc/autograd/graph_task.h`
- `torch/csrc/autograd/init.cpp`
- `torch/csrc/autograd/input_buffer.cpp`
- `torch/csrc/autograd/input_buffer.h`

**Overall Risk:** High

## Issues

### 1. Global mutable `PyObject*` variables set during module init
- **Category:** Static/global mutable `PyObject*`
- **Severity:** High
- **Confidence:** High
- **File:** `torch/csrc/autograd/init.cpp`
- **Line(s):** 110, 119, 126-127, 143
- **Description:** `THPAutograd_initExtension` sets four global `PyObject*` variables (`THPVariableClass`, `THPFunctionClass`, `THPGradientEdgeClass`, `ParameterClass`) without synchronization. Under free-threading, if two threads import the module concurrently (or one thread reads while another is initializing), there is a data race on these globals. With the GIL, module import was serialized, so this was safe.
- **Suggested Fix:** These are typically set once during module initialization. Use `std::atomic<PyObject*>` or `std::once_flag`/`std::call_once` to ensure initialization is safe. Alternatively, rely on Python's own import lock to serialize `THPAutograd_initExtension` calls (CPython free-threading already serializes module init via the import lock, so this may already be safe in practice, but it depends on how the values are read from other threads).

### 2. `Node::tensor_post_acc_grad_hooks()` returns reference to static local
- **Category:** Static mutable C++ state in Python-facing functions
- **Severity:** Medium
- **Confidence:** High
- **File:** `torch/csrc/autograd/function.h`
- **Line(s):** 549-553
- **Description:** The default implementation of `tensor_post_acc_grad_hooks()` returns a mutable reference (`std::unique_ptr<PostAccumulateGradHook>&`) to a `static` local variable (`empty`). If multiple threads call this method and any caller writes through the returned reference (e.g., assigning a hook), there is a data race on the shared static. Under the GIL, concurrent writes were serialized.
- **Suggested Fix:** This appears to be a fallback for nodes that don't override the method. If callers only ever read the returned `nullptr`, the race is benign (the value is never mutated). If any codepath writes to it, the static should be made thread-local or the design should be changed to return a value rather than a mutable reference. Audit callers to verify no writes happen through this reference for the base class implementation.

### 3. `ForwardGrad::empty()` reads `content_` without holding the mutex
- **Category:** Static mutable C++ state / data race
- **Severity:** Medium
- **Confidence:** Medium
- **File:** `torch/csrc/autograd/forward_grad.h`
- **Line(s):** 198-200
- **Description:** `ForwardGrad::empty()` reads `content_` without acquiring `mutex_`, while other methods (`set_value`, `reset`, `value`, `contains`) all acquire the lock before accessing `content_`. This creates a potential data race if `empty()` is called concurrently with `set_value()` or `reset()`. The `content_` member is a `std::unordered_map` which is not thread-safe for concurrent read/write.
- **Suggested Fix:** Acquire `mutex_` in `empty()`, consistent with the other accessors. Alternatively, document that `empty()` may only be called when no concurrent modifications are possible.

### 4. `PyList_GetItem` returns borrowed reference in `visit_tensors`
- **Category:** Borrowed references from Python containers
- **Severity:** Low
- **Confidence:** Medium
- **File:** `torch/csrc/autograd/init.cpp`
- **Line(s):** 1088
- **Description:** `PyList_GetItem(vals, i)` returns a borrowed reference. Under free-threading, if the list `vals` (created from `PyDict_Values(kwargs)`) is modified concurrently by another thread, the borrowed reference could be invalidated. However, `vals` is a locally created list not shared with other threads, so this is safe in practice. The `PyList_GET_ITEM` and `PyTuple_GET_ITEM` calls on `args`/`kwargs` are similarly low risk since these are function arguments with Python references held by the calling frame.
- **Suggested Fix:** No change needed. The `vals` list is locally owned and not shared. The borrowed references from `PyTuple_GET_ITEM` on args/kwargs are safe as long as the caller holds references (which Python function dispatch guarantees).

### 5. `Node::metadata()` lazy initialization of `anomaly_metadata_`
- **Category:** Lazy init patterns
- **Severity:** Low
- **Confidence:** Low
- **File:** `torch/csrc/autograd/function.cpp`
- **Line(s):** 49-53
- **Description:** `Node::metadata()` lazily initializes `anomaly_metadata_` with `if (!anomaly_metadata_) anomaly_metadata_ = ...`. This is a classic check-then-act race if called concurrently on the same `Node`. However, `metadata()` is typically called during graph construction (forward pass) where a `Node` is being built by a single thread, or during the backward pass where access is serialized by the engine's per-node mutex. The risk is low because the `Node` ownership model typically prevents concurrent access at this point.
- **Suggested Fix:** If concurrent access to the same Node is possible, protect with `mutex_` or use `std::call_once`. Given the existing usage patterns, this is likely safe.

## Summary

The most significant free-threading concern in this set of files is the initialization of global `PyObject*` pointers (`THPVariableClass`, `THPFunctionClass`, etc.) in `init.cpp`, though CPython's import lock may mitigate this in practice. The `ForwardGrad::empty()` missing lock is a real but contained race. The `tensor_post_acc_grad_hooks()` static local returning a mutable reference is a design issue that could manifest as a race if callers write through it. The engine and autograd graph machinery itself is already well-synchronized with explicit mutexes and atomics, reflecting its pre-existing multi-threaded design.
