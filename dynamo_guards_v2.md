# Free-Threading Audit: dynamo/guards.cpp and guards.h (v2)

## Architecture Summary

The dynamo guard system is a tree of C++ objects rooted at `RootGuardManager`.
Each `RootGuardManager` is attached to a cache entry in `ExtraState` on a Python
code object. When a compiled function is invoked, `eval_frame` iterates the
cache entry list and calls `run_root_guard_manager` on each to find a matching
compiled version.

**Module opt-in:** The module explicitly declares `Py_MOD_GIL_NOT_USED`
(line 6942), so under free-threading all code here runs without GIL protection.

**Concurrency-relevant data structures:**

| State | Scope | Accessed by |
|---|---|---|
| `dict_version_map` | process-global `unordered_map` | `dict_version_watch_callback` (any thread), `get_dict_version_unchecked` (guard eval) |
| `global_dict_version_id` | process-global `uint64_t` | same as above |
| `dict_to_guard_managers` | process-global `unordered_map` | `dict_recursive_tag_watch_callback` (any thread), `watch_dict_pointers` / `unwatch_all_saved_dict_pointers` (guard eval or destructor) |
| `RootGuardManager::_lock` | per-root `std::mutex` | serializes `check`/`check_verbose` per root |
| Per-`GuardManager` mutable state | per-node in guard tree | `_fail_count`, `_dict_tag`, `_accessors` ordering, `_dict_pointers` cache, `_disable_dict_tag_matching`, `_tensor_metadata_pointers`, `_tag_safe_entries` |
| `RelationalGuard` subclass state | shared via `shared_ptr` across cloned roots | `_is_first_call`, `_unique_tensors`, `_args_seen`, `_args_int`, `_args_float`, `StorageOverlapChecker` vectors |

**Locking model:** `RootGuardManager::check_nopybind_template` (line 3805)
releases the GIL, acquires `_lock`, then re-acquires the GIL. This serializes
all guard evaluation for a single root. However, the lock does NOT protect:
- Process-global state accessed from dict watcher callbacks
- GuardManager state accessed from watcher callbacks (which fire on any thread)
- Shared `RelationalGuard` instances reached from different roots

## SEVERE Issues

### 1. `dict_version_map` and `global_dict_version_id` concurrent access from watcher callbacks

- **Tier:** Tier 1
- **Shared state:** `static std::unordered_map<PyObject*, uint64_t> dict_version_map` (line 877), `static uint64_t global_dict_version_id` (line 880)
- **Writer(s):**
  - `dict_version_watch_callback` (line 881): fires on whatever thread mutates any watched dict. On `DEALLOCATED`, calls `dict_version_map.erase(dict)`. On other events, calls `dict_version_map[dict] = global_dict_version_id++`.
  - `get_dict_version_unchecked` (line 896): called during guard evaluation and during guard construction. Performs `dict_version_map.count(dict)`, `dict_version_map[dict] = global_dict_version_id++`, and `dict_version_map[dict]` (read).
- **Reader(s):** `get_dict_version_unchecked` from guard evaluation paths (`DICT_VERSION::check_nopybind`, `GuardManager::check_accessors_nopybind`, `GuardManager::record_dict_pointer`).
- **Race scenario:** Main thread evaluates guards on function `foo`, calling `get_dict_version_unchecked` which reads/writes `dict_version_map`. A data loader thread does a lazy `import` or writes to a module's `__dict__`, triggering `dict_version_watch_callback` which does `dict_version_map[dict] = global_dict_version_id++`. The concurrent structural modification of `std::unordered_map` (insert racing with insert, or insert racing with erase on `DEALLOCATED`) is undefined behavior -- hash table bucket corruption, infinite loops in bucket chains, or segfaults.
- **Consequence:** Crash (corrupted hash map internals).
- **Suggested fix:** Protect `dict_version_map` and `global_dict_version_id` with a mutex. The watcher callback must be able to acquire the lock (it is `noexcept`, so a `try_lock` with fallback is appropriate, or a simple mutex since these callbacks are short). Alternatively, replace with a concurrent hash map, or use `std::atomic<uint64_t>` for the counter and a locked map.

### 2. `dict_to_guard_managers` concurrent access from watcher callbacks and guard evaluation/destruction

- **Tier:** Tier 1
- **Shared state:** `std::unordered_map<PyObject*, std::list<GuardManager*>> dict_to_guard_managers` (line 1640)
- **Writer(s):**
  - `GuardManager::watch_dict_pointers` (line 3401): pushes `this` into `dict_to_guard_managers[dict_pointer]` during guard recording (called under `RootGuardManager::_lock` on the evaluating thread).
  - `GuardManager::unwatch_all_saved_dict_pointers` (line 3433): erases entries. Called from `disable_recursive_dict_tag_optimization` (line 3060), which is called from `dict_recursive_tag_watch_callback` (any thread), and from `~GuardManager()` (destructor, any thread).
  - `dict_recursive_tag_watch_callback` (line 4437): calls `dict_to_guard_managers.find(dict)`, iterates the result list, and calls `guard_manager->disable_recursive_dict_tag_optimization()` on each manager, which itself calls `unwatch_all_saved_dict_pointers()` which mutates `dict_to_guard_managers`.
- **Reader(s):** `dict_recursive_tag_watch_callback` (line 4443) does `dict_to_guard_managers.find(dict)` and iterates the list.
- **Race scenario (Tier 1):** Main thread evaluates guards, calling `watch_dict_pointers` which inserts into `dict_to_guard_managers`. Simultaneously, a data loader thread writes to a watched dict, triggering `dict_recursive_tag_watch_callback` which finds and iterates the same map. The concurrent `insert` (from `watch_dict_pointers`) and `find`+iterate (from the callback) on `std::unordered_map` is undefined behavior.
- **Race scenario (re-entrant mutation):** Even within a single callback invocation, the callback iterates `dict_to_guard_managers`, and for each guard manager calls `disable_recursive_dict_tag_optimization` -> `unwatch_all_saved_dict_pointers` which erases from the same map being iterated. The code copies `it->second` into a local (line 4445: `auto guard_managers = it->second`) which avoids iterator invalidation of the list, but `unwatch_all_saved_dict_pointers` can erase *other* keys from `dict_to_guard_managers`, invalidating iterators of the outer loop. In the callback code, only one key is looked up, so the current single-key pattern avoids the self-invalidation, but the cross-thread race remains.
- **Consequence:** Crash (corrupted hash map internals, dangling iterator dereference).
- **Suggested fix:** Protect `dict_to_guard_managers` with a global mutex. The callback (line 4437) must acquire the lock, as must `watch_dict_pointers` and `unwatch_all_saved_dict_pointers`. Since `disable_recursive_dict_tag_optimization` is called from within the callback while potentially holding the lock, care must be taken to avoid deadlock -- either use a recursive mutex, or restructure so the callback collects the set of managers to disable, releases the lock, then disables them.

### 3. `dict_recursive_tag_watch_callback` calls `disable_recursive_dict_tag_optimization` which mutates GuardManager state without the root lock

- **Tier:** Tier 1
- **Shared state:** `GuardManager::_disable_dict_tag_matching` (bool), `GuardManager::_dict_pointers` (unordered_map), `dict_to_guard_managers` (global map)
- **Writer(s):** `dict_recursive_tag_watch_callback` (line 4437) fires on any thread when a watched dict is mutated. It calls `guard_manager->disable_recursive_dict_tag_optimization()` (line 4449), which calls `unwatch_all_saved_dict_pointers()` (line 3433) that iterates and mutates `_dict_pointers`, then sets `_disable_dict_tag_matching = true`.
- **Reader(s):** `GuardManager::check_nopybind` (line 3213) reads `_disable_dict_tag_matching`, iterates `_dict_pointers`, reads `_dict_callback_installed` -- all under `RootGuardManager::_lock` on the evaluating thread.
- **Race scenario:** Main thread is evaluating guards, holding `RootGuardManager::_lock`, and is inside `GuardManager::check_nopybind` at line 3277 (`if (_dict_pointers.find(value) != _dict_pointers.end())`), iterating `_dict_pointers`. A data loader thread modifies a watched dict, triggering `dict_recursive_tag_watch_callback`, which calls `disable_recursive_dict_tag_optimization` on the SAME `GuardManager`, entering `unwatch_all_saved_dict_pointers` which mutates `_dict_pointers`. The watcher callback does NOT hold `RootGuardManager::_lock`. The concurrent read (main thread) and structural modification (callback thread) of `_dict_pointers` (an `unordered_map`) is undefined behavior.
- **Consequence:** Crash (corrupted map internals, dangling iterator).
- **Suggested fix:** Make the watcher callback only set an atomic flag (e.g., `std::atomic<bool> _dict_tag_invalidated`), and defer the actual unwatching (`unwatch_all_saved_dict_pointers`) to the next guard evaluation which already holds `_lock`. The guard evaluation would check the atomic flag at the start and call the cleanup if needed.

### 4. `disable_dict_tag_matching_callback` (weakref callback) dereferences `GuardManager*` concurrently with destruction

- **Tier:** Tier 1
- **Shared state:** `GuardManager*` stored inside a `PyCapsule` in `_tag_safe_entries`, and the `GuardManager` object itself.
- **Writer(s):** `~GuardManager()` calls `cleanup_tag_safe_entries()` (line 2957) which iterates `_tag_safe_entries` and calls `PyCapsule_SetName(e.cap, "DeadGuardManager")` then `Py_CLEAR(e.wr)`.
- **Reader(s):** `disable_dict_tag_matching_callback` (line 3340) is registered as a weakref callback. When the weakly-referenced Python object dies, this callback fires on whatever thread triggers GC. It calls `PyCapsule_IsValid(self_capsule, "GuardManager*")` then dereferences the pointer.
- **Race scenario:** Thread A destroys a `GuardManager` -- the destructor enters `cleanup_tag_safe_entries` and has invalidated entries [0..k-1] but not yet entry [k]. Thread B triggers GC on the Python object referenced by entry [k]'s weakref, firing `disable_dict_tag_matching_callback`. The callback sees a valid capsule (name still "GuardManager*") and dereferences the pointer. But the `GuardManager` destructor is actively running on Thread A, and the object may be partially destroyed. Under free-threading, weakref callbacks fire without GIL serialization.
- **Consequence:** Use-after-free or accessing partially-destroyed object.
- **Suggested fix:** Use an `std::atomic<bool>` in a shared control block (or inside the GuardManager) that both the destructor and the callback check/set atomically, instead of relying on `PyCapsule` name as a synchronization mechanism. Or ensure the weakref callback and the destructor are ordered (e.g., by dropping the weakrefs first, which should fire callbacks synchronously in CPython before the destructor proceeds).

## Significant Issues

### 5. Shared `RelationalGuard` instances across cloned `RootGuardManager`s race on mutable guard state

- **Tier:** Tier 2
- **Shared state:** `RelationalGuard` subclass fields: `OBJECT_ALIASING::_is_first_call`/`_first_tensor`, `NO_TENSOR_ALIASING::_unique_tensors`, `SYMBOLIC_SHAPE_GUARD::_args_seen`/`_args_int`/`_args_float`, `STORAGE_OVERLAPPING` via shared `StorageOverlapChecker`
- **Writer(s):** `check_nopybind` on these guards mutates state on every call (they accumulate state across multiple invocations within one evaluation). `reset_state` clears it at the end.
- **Reader(s):** Same methods. State is both read and written during guard evaluation.
- **Race scenario:** `clone_manager` (line 3937) creates a new `RootGuardManager` that SHARES `RelationalGuard` instances via `shared_ptr` (line 3084: `cloned_mgr->_leaf_guards.emplace_back(guard)` copies the shared_ptr, not the guard). If the original root and cloned root are on different cache entries (or different code objects), they have different `_lock` mutexes. Thread A evaluates the original root (holding root A's lock), setting `SYMBOLIC_SHAPE_GUARD::_args_seen = 1`. Thread B evaluates the cloned root (holding root B's lock), also setting `_args_seen = 1`. The shared guard's mutable state is corrupted by the interleaved writes.
- **Consequence:** Incorrect guard evaluation (wrong compiled code executed), or in the case of `StorageOverlapChecker`, race on `_overlapping`/`_non_overlapping` vectors with `Py_INCREF`/`Py_DECREF` which could crash.
- **Suggested fix:** Deep-copy `RelationalGuard` instances during cloning instead of sharing them. Each `RootGuardManager` should own independent copies.

### 6. `_accessors` sort and `_fail_count` mutation unprotected when `check` is called from Python on non-root managers

- **Tier:** Tier 2
- **Shared state:** `GuardManager::_accessors` (vector), `GuardManager::_fail_count` (int64_t)
- **Writer(s):** `check_accessors_nopybind` (line 3537) sorts `_accessors` on failure. `check_leaf_guards_nopybind` (line 3485) and `check_accessors_nopybind` (line 3517) increment `_fail_count`.
- **Reader(s):** Same methods, plus `fail_count()` (line 3615) used in the sort comparator.
- **Race scenario:** Within a single root, the `_lock` mutex serializes access. But `GuardManager` exposes `check` to Python, and non-root `GuardManager` objects can be accessed from Python for debugging. If two threads call `check` on the same non-root `GuardManager` (or one calls it while guard evaluation is in progress on another thread), the concurrent sort of `_accessors` is undefined behavior (concurrent moves of `unique_ptr` elements). Additionally, `_fail_count++` is a non-atomic read-modify-write.
- **Consequence:** Corrupted `_accessors` vector (crash), or benign stale count.
- **Suggested fix:** Mark `_fail_count` as `std::atomic<int64_t>`. For `_accessors` sorting, the risk is low since Python-side debugging calls are rare, but documenting that `check` on non-root managers is not thread-safe, or routing all calls through the root lock, would be prudent.

### 7. `GuardManager::_dict_tag` updated without synchronization across roots watching the same dict

- **Tier:** Tier 2
- **Shared state:** `GuardManager::_dict_tag` (uint64_t, line 3709)
- **Writer(s):** `check_accessors_nopybind` (line 3547): `_dict_tag = new_tag;` -- written at the end of successful accessor checks.
- **Reader(s):** `check_accessors_nopybind` (line 3507): `matches_dict_tag = (new_tag == _dict_tag);` -- read at the start.
- **Race scenario:** This is protected by `RootGuardManager::_lock` for a given root, so concurrent evaluations of the same root are safe. However, if the same `GuardManager` instance were reachable from two different roots (not the current architecture for non-leaf managers, but worth noting), the read and write would race. Under the current architecture, this is safe because non-root `GuardManager` instances are owned (via `unique_ptr`) by a single root.
- **Consequence:** Benign stale read (wrong tag match decision, causing unnecessary full traversal).
- **Note:** This is safe under the current ownership model but fragile if the architecture changes.

## Minor Issues

### 8. `make_guard_manager` static local initialization on pre-pybind 2.13

- **Tier:** Tier 2
- **Shared state:** `static py::object` variables in the `#else` branch (lines 4480-4485)
- **Writer(s):** First call to `make_guard_manager` initializes these.
- **Reader(s):** All subsequent calls.
- **Race scenario:** On the `IS_PYBIND_2_13_PLUS` path, `gil_safe_call_once_and_store` is used, which is thread-safe. On older pybind, C++ static local initialization guarantees thread-safe init, but `py::module_::import` called during init acquires Python's import lock and the GIL. Under free-threading, if two threads race to init simultaneously, the C++ runtime blocks one while the other does the import. This is safe as long as no circular dependency exists.
- **Consequence:** Unlikely deadlock if circular import dependency exists.
- **Suggested fix:** Ensure the pybind 2.13+ path is always used in free-threading builds.

### 9. `global_dict_version_id` non-atomic increment

- **Tier:** Tier 1
- **Shared state:** `static uint64_t global_dict_version_id = 1` (line 880)
- **Writer(s):** `dict_version_watch_callback` (line 889): `dict_version_map[dict] = global_dict_version_id++`. `get_dict_version_unchecked` (line 903): `dict_version_map[dict] = global_dict_version_id++`.
- **Reader(s):** Same increment expressions.
- **Race scenario:** Even if the `dict_version_map` is protected by a mutex (per fix for issue #1), the `global_dict_version_id++` is a non-atomic read-modify-write. Two threads could read the same value, both increment to the same next value, and assign the same "version" to two different dicts. This violates the invariant that each dict mutation gets a unique version ID.
- **Consequence:** Two dicts could end up with the same version, causing a guard to incorrectly consider a dict unchanged. This would cause wrong compiled code to execute (incorrect guard pass).
- **Suggested fix:** Make `global_dict_version_id` a `std::atomic<uint64_t>` and use `fetch_add(1)`.

### 10. `StorageOverlapChecker` reference counting races when shared across cloned roots

- **Tier:** Tier 2
- **Shared state:** `StorageOverlapChecker::_overlapping` and `_non_overlapping` vectors of `PyObject*`
- **Writer(s):** `add` calls `Py_INCREF`, `reset` calls `Py_DECREF`, `_overlapping`/`_non_overlapping` vectors are mutated.
- **Reader(s):** `maybe_check` reads the vectors.
- **Race scenario:** `StorageOverlapChecker` is shared via `shared_ptr` through `STORAGE_OVERLAPPING` guards, which are `RelationalGuard`s shared across cloned roots (issue #5). If two threads evaluate guards on different roots sharing the same checker, they race on `add`/`reset`/`maybe_check`.
- **Consequence:** Corrupted vectors, incorrect reference counts.
- **Suggested fix:** Addressed by fixing issue #5 (deep-copy relational guards during cloning).
