# `THPModule_initNames` static `names` vector concurrent access

- **Status:** Open
- **Severity:** Minor
- **Component:** Module.cpp

- **Shared state:** `names` -- a `static std::vector<std::string>` inside
  `THPModule_initNames()` (Module.cpp:165). This vector stores class name
  strings so that the `const char*` pointers assigned to `type->tp_name`
  remain valid.
- **Writer(s):**
  - `THPModule_initNames()` calls `names.emplace_back(...)` (line 186) and
    `names.reserve(...)` (line 173). This is called from Python during module
    init.
- **Reader(s):**
  - The `type->tp_name` pointers obtained from `names.back().c_str()` are
    stored on PyTypeObjects and read by Python whenever it formats type names
    (e.g., error messages, `type(x).__name__`). A `push_back` that
    reallocates would invalidate all existing `tp_name` pointers.
- **Race scenario:** Same as the `addDocStr` issue -- this is an init-time
  operation protected by Python's import lock in practice, but the
  `c_str()` invalidation pattern is dangerous if concurrent access occurs.
  Two threads racing on `_initNames` (unlikely but possible with manual
  imports) could cause `tp_name` to point to freed memory.
- **Suggested fix:** Use `std::deque<std::string>` instead of
  `std::vector<std::string>` to avoid pointer invalidation on growth.
  Alternatively, accept the import-lock protection as sufficient.
