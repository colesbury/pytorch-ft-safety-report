# `disabled_torch_function` / `disabled_torch_dispatch` global PyObject* pointers

- **Status:** Open
- **Severity:** Significant
- **Component:** disable_torch_function
- **Source reports:** [utils__part1.md](../utils__part1.md)

- **Shared state:** `disabled_torch_function` and `disabled_torch_dispatch`
  -- file-scope static `PyObject*` pointers in
  `torch/csrc/utils/disable_torch_function.cpp` (lines 10-11).
- **Writer(s):**
  - `set_disabled_torch_function_impl()` (line 22-24): sets the pointer.
  - `set_disabled_torch_dispatch_impl()` (line 30-32): sets the pointer.
  - Both are called from Python during module initialization.
- **Reader(s):**
  - `disabled_torch_function_impl()` (line 18-20): returns the raw pointer.
  - `disabled_torch_dispatch_impl()` (line 26-28): returns the raw pointer.
  - `has_torch_function_attr()` (line 312-315 in the same file): reads
    `disabled_torch_function` and compares it with the result of
    `PyObject_FastGetAttrString`. This is called on every op dispatch
    path that checks for `__torch_function__` overrides.
  - `dispatch_on_subclass()` in `python_arg_parser.cpp` (line 347):
    reads `disabled_torch_dispatch_impl()`.
- **Race scenario:** If `set_disabled_torch_function_impl` is called
  from one thread while another thread is in `has_torch_function_attr`
  comparing the old pointer value, the reader may see a torn write
  (on platforms where pointer writes are not atomic) or a stale value.
  In practice, these pointers are set once during module init and never
  changed, so the realistic risk is that a thread calling
  `has_torch_function_attr` before module init completes reads a null
  pointer and incorrectly concludes the object has `__torch_function__`
  (since `nullptr != attr.ptr()` would be true for any non-null attr).
  This is more of a correctness issue than a crash, but it happens on
  a very hot path (every tensor operation).
- **Suggested fix:** Since these are set once during init and then
  read-only, use `std::atomic<PyObject*>` with relaxed stores during
  init and acquire loads during reads to ensure visibility. Or simply
  document that they must be set before any concurrent access begins
  (which Python's import lock guarantees).
