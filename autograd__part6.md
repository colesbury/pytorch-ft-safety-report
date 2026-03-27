# Free-Threading Audit: autograd__part6

**Files:**
- `torch/csrc/autograd/symbolic.h`
- `torch/csrc/autograd/variable.cpp`
- `torch/csrc/autograd/variable.h`
- `torch/csrc/autograd/variable_info.cpp`
- `torch/csrc/autograd/variable_info.h`

**Overall Risk:** Low

## Issues

### 1. Global mutable `singleton_undefined_tensor`
- **Category:** Static/global mutable state
- **Severity:** Low
- **Confidence:** Medium
- **File:** `torch/csrc/autograd/variable.cpp`
- **Line(s):** 150
- **Description:** `singleton_undefined_tensor` is a file-scope `at::Tensor` that is default-constructed (empty/undefined). It is returned by const reference from `ConcreteAutogradMetaFactory::undefined_tensor()`. Because `at::Tensor` internally holds an `intrusive_ptr` (which is null here), reads are safe. However, if any code path ever mutates this tensor (e.g., accidentally assigning to it), there would be a data race. Currently no mutation is observable in these files, so the practical risk is low. Under free-threading the static initialization itself is safe (zero-init + constant init), but the type is technically non-trivially-destructible, which is a minor concern during shutdown only.
- **Suggested Fix:** Declare as `const at::Tensor` or wrap in a function returning a `const` reference to make immutability explicit and prevent accidental mutation.

### 2. Global mutable `singleton_shared_ptr` returned by reference from `grad_fn()`
- **Category:** Static/global mutable state
- **Severity:** Low
- **Confidence:** Medium
- **File:** `torch/csrc/autograd/variable.cpp`
- **Line(s):** 649
- **Description:** `singleton_shared_ptr` is a file-scope `std::shared_ptr<Node>` (default-constructed to nullptr) returned by const reference when a tensor has no autograd meta. The reference is to a global, so if a caller ever stored a non-const reference and wrote to it, or if the shared_ptr's reference count were modified by a copy occurring concurrently with a read of the control block, there could be a race. In practice, callers receive `const std::shared_ptr<Node>&` and the pointer is always null, so the practical risk is low.
- **Suggested Fix:** Declare as `const std::shared_ptr<torch::autograd::Node> singleton_shared_ptr;` or use a local static in the function to make the intent clear and prevent accidental mutation.

### 3. Global mutable `singleton_string` returned by reference from `name()`
- **Category:** Static/global mutable state
- **Severity:** Low
- **Confidence:** Low
- **File:** `torch/csrc/autograd/variable.cpp`
- **Line(s):** 635
- **Description:** Same pattern as the above two: a file-scope `std::string` (default empty) returned by const reference. No mutation is observed, but the non-const declaration leaves it vulnerable to accidental writes.
- **Suggested Fix:** Declare as `const std::string singleton_string;`.

## Summary

These files are predominantly C++ autograd internals that operate on per-tensor state (protected by per-tensor `mutex_` where needed) rather than Python objects. The three global singletons (`singleton_undefined_tensor`, `singleton_shared_ptr`, `singleton_string`) are effectively read-only after static initialization but are not declared `const`, which is a minor defensive concern. No Python C API usage, no `PyObject*` globals, and no GIL-dependent patterns were found. The overall free-threading risk is low.
