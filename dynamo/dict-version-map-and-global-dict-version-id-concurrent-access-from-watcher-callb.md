# `dict_version_map` and `global_dict_version_id` concurrent access from watcher callbacks

- **Status:** Open
- **Severity:** SEVERE
- **Tier:** Tier 1
- **Component:** guards
- **Source report:** [dynamo_guards_v2.md](../dynamo_guards_v2.md)

- **Tier:** Tier 1
- **Shared state:** `static std::unordered_map<PyObject*, uint64_t> dict_version_map` (line 877), `static uint64_t global_dict_version_id` (line 880)
- **Writer(s):**
  - `dict_version_watch_callback` (line 881): fires on whatever thread mutates any watched dict. On `DEALLOCATED`, calls `dict_version_map.erase(dict)`. On other events, calls `dict_version_map[dict] = global_dict_version_id++`.
  - `get_dict_version_unchecked` (line 896): called during guard evaluation and during guard construction. Performs `dict_version_map.count(dict)`, `dict_version_map[dict] = global_dict_version_id++`, and `dict_version_map[dict]` (read).
- **Reader(s):** `get_dict_version_unchecked` from guard evaluation paths (`DICT_VERSION::check_nopybind`, `GuardManager::check_accessors_nopybind`, `GuardManager::record_dict_pointer`).
- **Race scenario:** Main thread evaluates guards on function `foo`, calling `get_dict_version_unchecked` which reads/writes `dict_version_map`. A data loader thread does a lazy `import` or writes to a module's `__dict__`, triggering `dict_version_watch_callback` which does `dict_version_map[dict] = global_dict_version_id++`. The concurrent structural modification of `std::unordered_map` (insert racing with insert, or insert racing with erase on `DEALLOCATED`) is undefined behavior -- hash table bucket corruption, infinite loops in bucket chains, or segfaults.
- **Consequence:** Crash (corrupted hash map internals).
- **Suggested fix:** Protect `dict_version_map` and `global_dict_version_id` with a mutex. The watcher callback must be able to acquire the lock (it is `noexcept`, so a `try_lock` with fallback is appropriate, or a simple mutex since these callbacks are short). Alternatively, replace with a concurrent hash map, or use `std::atomic<uint64_t>` for the counter and a locked map.
