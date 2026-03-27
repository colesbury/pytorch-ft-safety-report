# Deep Free-Threading Audit: dynamo/guards.cpp

## Architecture Summary

The dynamo guard system is a tree of C++ objects rooted at `RootGuardManager`. Each
`RootGuardManager` is attached to a `CacheEntry`, which is stored in an `ExtraState`
attached to a Python code object. When a function is called, `eval_frame` iterates
the `ExtraState`'s cache entry list and calls `run_root_guard_manager` on each entry
to find a cached compiled version.

**Critical concurrency model for free-threading:** Under free-threading, two threads
can execute the same function simultaneously. This means two threads can call
`run_root_guard_manager` on the *same* `RootGuardManager` instance concurrently,
because code objects (and their ExtraState) are shared across threads. Additionally,
dict watcher callbacks fire on whichever thread mutates a dict -- they are completely
asynchronous with respect to guard evaluation.

The module explicitly opts into free-threading via `PyUnstable_Module_SetGIL(m,
Py_MOD_GIL_NOT_USED)` (line 6942).

**Key shared mutable state:**
1. `dict_version_map` -- process-global `unordered_map` mapping dict pointers to version IDs
2. `global_dict_version_id` -- process-global counter
3. `dict_to_guard_managers` -- process-global map from dict pointers to lists of GuardManager*
4. `RootGuardManager::_lock` -- per-root mutex used by check/check_verbose
5. Per-GuardManager mutable state: `_fail_count`, `_dict_tag`, `_accessors` ordering, dict-tag caches

## SEVERE Issues

### 1. `dict_version_map` and `global_dict_version_id` are unprotected process-global state

- **Shared state:** `static std::unordered_map<PyObject*, uint64_t> dict_version_map` (line 877) and `static uint64_t global_dict_version_id` (line 880)
- **Writer(s):** `dict_version_watch_callback` (line 881) writes on every dict mutation of any watched dict. `get_dict_version_unchecked` (line 896) also writes (it inserts into the map and increments the counter when a dict is first seen).
- **Reader(s):** `get_dict_version_unchecked` reads the map from guard evaluation (called from `DICT_VERSION::check_nopybind`, `GuardManager::check_accessors_nopybind`, and `GuardManager::record_dict_pointer`). Guard evaluation happens from any thread calling the compiled function.
- **Race scenario:** Thread A is evaluating guards for function `foo` and calls `get_dict_version_unchecked` on a dict, which does `dict_version_map[dict] = global_dict_version_id++`. Simultaneously, Thread B mutates any watched dict, triggering `dict_version_watch_callback` which does `dict_version_map[dict] = global_dict_version_id++`. Both threads are concurrently reading and writing the same `unordered_map` without any synchronization. This is undefined behavior -- `unordered_map` is not thread-safe, and concurrent insert/erase/lookup can corrupt the hash table's internal bucket structure, causing crashes, infinite loops, or memory corruption.
- **Severity:** SEVERE
- **Suggested fix:** Protect `dict_version_map` and `global_dict_version_id` with a mutex, or use a concurrent hash map. The watcher callback path must be lock-free-friendly since it runs in arbitrary contexts; a `std::shared_mutex` (read-lock for lookups, write-lock for mutations) would work. Alternatively, under free-threading, consider using CPython's per-dict `ma_version_tag` equivalent if available, eliminating the need for the global map entirely.

### 2. `dict_to_guard_managers` is unprotected process-global state raced by watcher callbacks

- **Shared state:** `std::unordered_map<PyObject*, std::list<GuardManager*>> dict_to_guard_managers` (line 1640)
- **Writer(s):**
  - `GuardManager::watch_dict_pointers` (line 3401) pushes `this` into `dict_to_guard_managers[dict_pointer]` during guard recording (called from guard evaluation under `RootGuardManager::_lock`).
  - `GuardManager::unwatch_all_saved_dict_pointers` (line 3433) erases entries during guard manager destruction or when dict-tag optimization is disabled.
  - `dict_recursive_tag_watch_callback` (line 4437) reads and iterates the map when any watched dict is mutated.
- **Reader(s):** `dict_recursive_tag_watch_callback` (line 4443) does `dict_to_guard_managers.find(dict)` and then iterates the resulting list.
- **Race scenario:** Thread A is destroying a `GuardManager` (e.g., during cache eviction or Python GC), calling `unwatch_all_saved_dict_pointers` which erases from `dict_to_guard_managers`. Thread B is modifying a dict on a completely different code path, triggering `dict_recursive_tag_watch_callback` which is iterating the same `dict_to_guard_managers` map. The concurrent structural modification (erase during iteration/find) of `unordered_map` is undefined behavior and will crash.
- **Severity:** SEVERE
- **Suggested fix:** Protect `dict_to_guard_managers` with a global mutex. All three access sites (watch, unwatch, callback) need to hold the lock. The callback is `noexcept` so it must not throw; a `try_lock` with fallback to "do nothing" would be acceptable since the guard manager already disables the optimization on failure.

### 3. `RootGuardManager::_lock` does not protect child GuardManager state accessed during recursive dict-tag matching

- **Shared state:** `GuardManager::_dict_pointers`, `_tensor_pointers`, `_tensor_metadata_pointers`, `_disable_dict_tag_matching`, `_dict_callback_installed`, `_tag_safe_entries`, `_fail_count`, `_dict_tag`, `_accessors` ordering
- **Writer(s):** `GuardManager::check_nopybind` (line 3213) writes `_disable_dict_tag_matching`, calls `start_recording_dict_pointers`, calls `stop_recording_dict_pointers` which calls `stash_dict_pointers`/`stash_tensor_pointers`/`stash_tensor_metadata`, calls `register_weakref_callback`, calls `watch_dict_pointers`. `check_accessors_nopybind` sorts `_accessors` and writes `_dict_tag`. `check_leaf_guards_nopybind` increments `_fail_count`.
- **Reader(s):** Same methods, called concurrently on the same `GuardManager` from two threads evaluating the same function.
- **Race scenario:** The `RootGuardManager::_lock` mutex serializes access at the ROOT level. But `GuardManager::check_nopybind` for non-root managers (children in the tree) is called from within the locked region of one root, but the SAME `GuardManager` instance can also be reached from a DIFFERENT `RootGuardManager` (when guards are cloned via `clone_manager`, leaf guards are shared via `shared_ptr`). Furthermore, `dict_recursive_tag_watch_callback` calls `guard_manager->disable_recursive_dict_tag_optimization()` which calls `unwatch_all_saved_dict_pointers()` which mutates `_dict_pointers` -- this callback fires on ANY thread, outside of any lock.
- **Severity:** SEVERE
- **Suggested fix:** The `disable_recursive_dict_tag_optimization` path from watcher callbacks must either be protected by a lock or use atomic operations. Making `_disable_dict_tag_matching` an `std::atomic<bool>` would handle the simple flag case, but `unwatch_all_saved_dict_pointers` also mutates `_dict_pointers` (a map), so that entire method needs synchronization. One approach: make the watcher callback only set an atomic flag, and defer the actual unwatching to the next guard evaluation (which is already under `_lock`).


## Significant Issues

### 4. RelationalGuard state is shared across cloned RootGuardManagers and can race

- **Shared state:** `RelationalGuard` subclass instance fields: `OBJECT_ALIASING::_is_first_call`/`_first_tensor`, `NO_TENSOR_ALIASING::_unique_tensors`, `SYMBOLIC_SHAPE_GUARD::_args_seen`/`_args_int`/`_args_float`, `STORAGE_OVERLAPPING`'s `StorageOverlapChecker`
- **Writer(s):** `check_nopybind` methods on these guards mutate internal state on every invocation (they accumulate state across multiple calls within a single guard evaluation). `reset_state` clears the state.
- **Reader(s):** Same methods. The state is written during guard evaluation and reset at the end.
- **Race scenario:** When `clone_manager` is called (line 3937), the cloned `RootGuardManager` SHARES `RelationalGuard` instances via `shared_ptr` (line 3084 -- `cloned_mgr->_leaf_guards.emplace_back(guard)` copies the `shared_ptr`). If the original root and the cloned root are attached to different code objects (or different cache entries on the same code), two threads could evaluate guards simultaneously: Thread A evaluates the original root's guards, setting `SYMBOLIC_SHAPE_GUARD::_args_seen = 1` and writing `_args_int[0]`. Thread B evaluates the cloned root's guards, also writing `_args_seen` and `_args_int[0]`. The interleaved writes corrupt the guard state, leading to wrong guard decisions (accepting invalid code or rejecting valid code).
- **Note:** Even if both roots are on the same code object, the per-root `_lock` only serializes within one root, not across different roots sharing the same relational guard.
- **Severity:** Significant (likely causes incorrect guard evaluation, not necessarily a crash, but could lead to executing wrong compiled code)
- **Suggested fix:** Do not share `RelationalGuard` instances across cloned roots. Deep-copy them instead. Alternatively, make the relational guard state thread-local or per-evaluation.

### 5. `_accessors` vector sorting during guard evaluation is not safe under concurrent evaluation

- **Shared state:** `GuardManager::_accessors` (a `vector<unique_ptr<GuardAccessor>>`)
- **Writer(s):** `check_accessors_nopybind` (line 3537) calls `std::sort` on `_accessors` when a guard fails and it was not the first accessor.
- **Reader(s):** The same `check_accessors_nopybind` iterates `_accessors`. Since `RootGuardManager::_lock` serializes the root, concurrent access to the same accessor vector within one root is prevented. However, if a `GuardManager` is shared (via cloning with shared leaf guards -- though accessors are not shared by cloning), this is safe for the common case.
- **Race scenario:** Within a single `RootGuardManager`, the mutex prevents this. But the `_fail_count` field (line 3485, 3517) is a plain `int64_t` that is incremented without synchronization by `dict_recursive_tag_watch_callback`'s indirect path. More importantly, if `check_nopybind` is ever called on a `GuardManager` *outside* of the root lock (e.g., from Python-side debugging calls like `guard_manager.check(value)`), concurrent sorts would corrupt the vector.
- **Severity:** Significant (the root lock mitigates the hot-path case, but the Python-facing `check` method on non-root managers has no locking)
- **Suggested fix:** Either ensure all check paths go through the root lock, or make the sort use atomic operations / a separate lock.

### 6. `dict_version_watch_callback` for the version-map watcher races with `get_dict_version_unchecked`

- **Shared state:** `dict_version_map` (same as issue #1 but focusing on the specific version watcher)
- **Writer(s):** `dict_version_watch_callback` (line 881) -- fires on `PyDict_EVENT_DEALLOCATED` calling `dict_version_map.erase(dict)`, or on other events calling `dict_version_map[dict] = global_dict_version_id++`
- **Reader(s):** `get_dict_version_unchecked` (line 896) calls `dict_version_map.count(dict)` and `dict_version_map[dict]`
- **Race scenario:** Thread A calls a compiled function whose guard includes `DICT_VERSION`. This calls `get_dict_version_unchecked`, which does `dict_version_map.count(dict)`. Thread B mutates any dict (or a dict gets GC'd), triggering `dict_version_watch_callback` which calls `dict_version_map.erase(dict)`. The `count` and `erase` happen concurrently on the same `unordered_map` -- UB.
- **Severity:** Significant (this is really the same root cause as #1, listed separately to emphasize that the version-map watcher and the tag watcher are two independent watcher IDs with the same problem)
- **Suggested fix:** Same as #1.


## Minor Issues

### 7. `_fail_count` is a non-atomic `int64_t` read/written from the watcher callback path

- **Shared state:** `GuardManager::_fail_count` (line 3670)
- **Writer(s):** `check_leaf_guards_nopybind` (line 3485) and `check_accessors_nopybind` (line 3517) increment `_fail_count`.
- **Reader(s):** `fail_count()` (line 3615) returns the value; used in the sort comparator (line 3542).
- **Race scenario:** Under the current architecture, `_fail_count` is always accessed under `RootGuardManager::_lock` for the hot path. But `dict_recursive_tag_watch_callback` -> `disable_recursive_dict_tag_optimization` could transitively access a path that doesn't hold the lock. In practice the callback only sets a bool flag and doesn't touch `_fail_count`, so this is currently safe but fragile.
- **Severity:** Minor (benign today due to the lock, but a stale read is not dangerous -- it only affects sort ordering for fail-fast optimization)
- **Suggested fix:** Make `_fail_count` `std::atomic<int64_t>` for defense in depth.

### 8. `make_guard_manager` static local initialization is racy on pre-pybind 2.13

- **Shared state:** The `static py::object` variables in the `#else` branch of `make_guard_manager` (lines 4480-4485)
- **Writer(s):** First call to `make_guard_manager` initializes these statics.
- **Reader(s):** All subsequent calls read them.
- **Race scenario:** On the `IS_PYBIND_2_13_PLUS` path, `gil_safe_call_once_and_store` is used which is thread-safe. On the older pybind path, `static py::object` initialization relies on C++ static local thread-safety (guaranteed by C++11), but the `py::module_::import` call requires the GIL. Under free-threading, two threads could both attempt the initialization simultaneously -- C++ guarantees only one executes the initializer, but the other thread blocks, and if the blocked thread holds a resource the initializer needs, deadlock could occur. However, since this is module-level import, it typically happens during import time when only one thread is active.
- **Severity:** Minor (unlikely in practice since imports are typically single-threaded, and the 2.13+ path is likely used in modern builds)
- **Suggested fix:** Ensure the pybind 2.13+ path is always used, or wrap the old path in a `call_once`.

### 9. `GuardManager::disable_dict_tag_matching_callback` dereferences a `GuardManager*` from a PyCapsule without synchronization

- **Shared state:** The `GuardManager*` stored in a PyCapsule, and the `GuardManager` object's lifetime
- **Writer(s):** `~GuardManager()` calls `cleanup_tag_safe_entries` which sets the capsule name to "DeadGuardManager" to invalidate it.
- **Reader(s):** `disable_dict_tag_matching_callback` (line 3340) is a weakref callback that checks `PyCapsule_IsValid` and then dereferences the pointer.
- **Race scenario:** Thread A is destroying a `GuardManager`, has entered the destructor, and is partway through `cleanup_tag_safe_entries`. Thread B triggers a weakref callback for a different entry in `_tag_safe_entries` that hasn't been invalidated yet -- it reads a valid capsule and dereferences the `GuardManager*` which is being destroyed. The check `PyCapsule_IsValid(self_capsule, "GuardManager*")` and the subsequent `PyCapsule_GetPointer` are not atomic with respect to `PyCapsule_SetName` in the destructor.
- **Severity:** Minor (the destructor and the weakref callback both require the GIL in GIL-enabled Python, but under free-threading, weakref callbacks can fire concurrently with object destruction on different threads)
- **Suggested fix:** Use an `std::atomic<bool>` flag on the GuardManager (or in a shared control block) that is checked by the callback and set by the destructor, rather than relying on PyCapsule name mutation for synchronization.

### 10. `StorageOverlapChecker` reference counting of PyObject* via raw `Py_INCREF`/`Py_DECREF`

- **Shared state:** `StorageOverlapChecker::_overlapping` and `_non_overlapping` vectors of `PyObject*`
- **Writer(s):** `add` calls `Py_INCREF`, `reset` calls `Py_DECREF`
- **Reader(s):** `maybe_check` reads the vectors and unpacks tensors
- **Race scenario:** `StorageOverlapChecker` is shared via `shared_ptr` between two `STORAGE_OVERLAPPING` guard instances which are themselves shared via `shared_ptr` across cloned `RootGuardManager`s. If two threads evaluate guards on different roots that share the same checker, both could call `add`/`reset`/`maybe_check` concurrently, racing on the vectors and the reference counts. However, in practice the `RootGuardManager::_lock` serializes all guard evaluation per-root, and the checker accumulates state within a single evaluation, so this is only a problem if the checker is shared across roots (via cloning with shared relational guards, as described in issue #4).
- **Severity:** Minor (contingent on issue #4 being triggered)
- **Suggested fix:** Addressed by fixing issue #4 (don't share relational guards across roots).
