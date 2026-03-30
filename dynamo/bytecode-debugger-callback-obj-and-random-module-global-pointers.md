# `bytecode_debugger_callback_obj` and `random_module` global pointers

- **Status:** Open
- **Severity:** Minor
- **Tier:** Tier 2
- **Component:** eval_frame
- **Source report:** [dynamo_eval_frame_v2.md](../dynamo_eval_frame_v2.md)

- **Shared state:** `bytecode_debugger_callback_obj` (line 25) and
  `random_module` (line 220) -- file-scope `PyObject*` pointers in
  `eval_frame_cpp.cpp`.
- **Writer(s):**
  - `set_bytecode_debugger_callback()` uses `Py_XSETREF` to swap.
  - `get_random_module()` lazily initializes `random_module`.
- **Reader(s):**
  - `get_bytecode_debugger_callback()` reads `bytecode_debugger_callback_obj`.
  - `PreserveGlobalState` constructor/destructor calls `get_random_module()`.
- **Race scenario:** For `bytecode_debugger_callback_obj`: similar to
  issue #8 but this is a debugging feature rarely used in production. For
  `random_module`: lazy init TOCTOU -- two threads could both see `nullptr`
  and both import, leaking one reference. The import itself is serialized by
  Python's import lock, so no crash, just a leaked reference.
- **Tier:** **Tier 2**.
- **Suggested fix:** For `random_module`, use C++11 `static` local for
  thread-safe lazy init. For `bytecode_debugger_callback_obj`, use
  `std::atomic<PyObject*>` with appropriate refcounting.
