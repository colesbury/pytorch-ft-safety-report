# `THPModule_addDocStr` static `all_docs` vector concurrent access

- **Status:** Open
- **Severity:** Minor
- **Component:** Module.cpp

- **Shared state:** `all_docs` -- a `static std::vector<std::string>` inside
  `THPModule_addDocStr()` (Module.cpp:436). This vector stores docstrings so
  that the `const char*` pointers assigned to function/method/type `ml_doc`
  fields remain valid for the lifetime of the process.
- **Writer(s):**
  - `THPModule_addDocStr()` calls `all_docs.push_back(...)` (line 445) each
    time a docstring is added. This is called from Python during module
    initialization (e.g., `torch._C._add_docstr(func, "...")` in
    `torch/__init__.py`).
- **Reader(s):**
  - The `const char*` pointers obtained from `all_docs.back().c_str()` are
    stored in function/method descriptors and read whenever Python accesses
    `__doc__` on those objects. However, a `push_back` that triggers
    reallocation would invalidate all existing `c_str()` pointers, which are
    stored in `ml_doc` fields across the codebase.
- **Race scenario:** In practice this is only called during module import, which
  is serialized by Python's import lock. However, if two threads somehow
  race on `_add_docstr` calls (e.g., via manual `importlib` usage), a
  `push_back` that reallocates the vector would invalidate all previously
  stored `c_str()` pointers, causing use-after-free when any thread reads
  `__doc__` from a previously documented function.
  The risk is low because this is an init-time operation, but the
  `c_str()` invalidation pattern is particularly dangerous because the
  corruption is silent and delayed.
- **Suggested fix:** Use `std::deque<std::string>` instead of
  `std::vector<std::string>` -- `deque::push_back` does not invalidate
  existing element references/pointers. Alternatively, since this is
  init-only, no fix may be needed if Python's import lock is sufficient.
