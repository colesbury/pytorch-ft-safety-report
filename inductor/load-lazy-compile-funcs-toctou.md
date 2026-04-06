# TOCTOU on `loadLazyCompileFuncs()` statics

- **Status:** Open
- **Severity:** Significant
- **Component:** cpp_wrapper/lazy_triton_compile.h

- **Shared state:** `triton_lazy_compile_module` and four other static
  `PyObject*` pointers in `lazy_triton_compile.h`
- **Writer:** `loadLazyCompileFuncs()` checks null, then stores to five pointers
  without synchronization.
- **Race scenario:** Currently called from static initializer (serialized by
  `dlopen`), but the pattern is fragile. If ever called from a runtime path,
  two concurrent calls would race on the stores.
- **Consequence:** Stale or partially-initialized function pointers, crash if
  called through them.
- **Suggested fix:** `std::call_once`.
