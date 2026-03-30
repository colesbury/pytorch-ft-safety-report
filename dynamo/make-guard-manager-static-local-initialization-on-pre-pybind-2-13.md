# `make_guard_manager` static local initialization on pre-pybind 2.13

- **Status:** Open
- **Severity:** Minor
- **Tier:** Tier 2
- **Component:** guards
- **Source report:** [dynamo_guards_v2.md](../dynamo_guards_v2.md)

- **Tier:** Tier 2
- **Shared state:** `static py::object` variables in the `#else` branch (lines 4480-4485)
- **Writer(s):** First call to `make_guard_manager` initializes these.
- **Reader(s):** All subsequent calls.
- **Race scenario:** On the `IS_PYBIND_2_13_PLUS` path, `gil_safe_call_once_and_store` is used, which is thread-safe. On older pybind, C++ static local initialization guarantees thread-safe init, but `py::module_::import` called during init acquires Python's import lock and the GIL. Under free-threading, if two threads race to init simultaneously, the C++ runtime blocks one while the other does the import. This is safe as long as no circular dependency exists.
- **Consequence:** Unlikely deadlock if circular import dependency exists.
- **Suggested fix:** Ensure the pybind 2.13+ path is always used in free-threading builds.
