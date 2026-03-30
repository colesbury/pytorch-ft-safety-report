# `dict_to_guard_managers` concurrent access from watcher callbacks and guard evaluation/destruction

- **Status:** Fix pending — [pytorch/pytorch#178703](https://github.com/pytorch/pytorch/pull/178703)
- **Severity:** SEVERE
- **Tier:** Tier 1
- **Component:** guards
- **Source report:** [dynamo_guards_v2.md](../dynamo_guards_v2.md)

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
