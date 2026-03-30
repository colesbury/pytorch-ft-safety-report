# `dict_recursive_tag_watch_callback` calls `disable_recursive_dict_tag_optimization` which mutates GuardManager state without the root lock

- **Status:** Fix pending — [pytorch/pytorch#178703](https://github.com/pytorch/pytorch/pull/178703)
- **Severity:** SEVERE
- **Tier:** Tier 1
- **Component:** guards
- **Source report:** [dynamo_guards_v2.md](../dynamo_guards_v2.md)

- **Tier:** Tier 1
- **Shared state:** `GuardManager::_disable_dict_tag_matching` (bool), `GuardManager::_dict_pointers` (unordered_map), `dict_to_guard_managers` (global map)
- **Writer(s):** `dict_recursive_tag_watch_callback` (line 4437) fires on any thread when a watched dict is mutated. It calls `guard_manager->disable_recursive_dict_tag_optimization()` (line 4449), which calls `unwatch_all_saved_dict_pointers()` (line 3433) that iterates and mutates `_dict_pointers`, then sets `_disable_dict_tag_matching = true`.
- **Reader(s):** `GuardManager::check_nopybind` (line 3213) reads `_disable_dict_tag_matching`, iterates `_dict_pointers`, reads `_dict_callback_installed` -- all under `RootGuardManager::_lock` on the evaluating thread.
- **Race scenario:** Main thread is evaluating guards, holding `RootGuardManager::_lock`, and is inside `GuardManager::check_nopybind` at line 3277 (`if (_dict_pointers.find(value) != _dict_pointers.end())`), iterating `_dict_pointers`. A data loader thread modifies a watched dict, triggering `dict_recursive_tag_watch_callback`, which calls `disable_recursive_dict_tag_optimization` on the SAME `GuardManager`, entering `unwatch_all_saved_dict_pointers` which mutates `_dict_pointers`. The watcher callback does NOT hold `RootGuardManager::_lock`. The concurrent read (main thread) and structural modification (callback thread) of `_dict_pointers` (an `unordered_map`) is undefined behavior.
- **Consequence:** Crash (corrupted map internals, dangling iterator).
- **Suggested fix:** Make the watcher callback only set an atomic flag (e.g., `std::atomic<bool> _dict_tag_invalidated`), and defer the actual unwatching (`unwatch_all_saved_dict_pointers`) to the next guard evaluation which already holds `_lock`. The guard evaluation would check the atomic flag at the start and call the cleanup if needed.
