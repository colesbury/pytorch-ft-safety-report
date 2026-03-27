# Deep Free-Threading Audit Summary: `torch/csrc/dynamo/`

All three dynamo modules declare `Py_MOD_GIL_NOT_USED`, explicitly opting into free-threading, but the internal state is not prepared for concurrent access.

## SEVERE (will crash or corrupt under free-threading)

| # | Component | Issue | Root Cause |
|---|-----------|-------|------------|
| 1 | guards.cpp | `dict_version_map` + `global_dict_version_id` — unprotected global `unordered_map` mutated from dict watcher callbacks (any thread) while guard evaluation reads it. Concurrent insert/erase = hash table corruption → crash. | No synchronization on process-global map |
| 2 | guards.cpp | `dict_to_guard_managers` — same pattern. Watcher callback iterates while destructor erases → UB. | No synchronization on process-global map |
| 3 | guards.cpp | `dict_recursive_tag_watch_callback` mutates `GuardManager` fields (`_dict_pointers`, `_disable_dict_tag_matching`) from arbitrary threads, outside `RootGuardManager::_lock`. | Watcher callback has no lock |
| 4 | eval_frame | `ExtraState::cache_entry_list` — two threads calling the same function race on `std::list` iteration vs insertion/splice. No synchronization on ExtraState at all. | Code objects (and ExtraState) are shared across threads |
| 5 | eval_frame | Double-init of ExtraState — two threads both see `extra == nullptr`, both call `init_and_set_extra_state`, second asserts and crashes. | Check-then-act without lock |
| 6 | eval_frame | Concurrent compilation — two threads both get cache miss, both compile, both mutate `frame_state` (a `py::dict`) concurrently → dict corruption. | No compilation lock per code object |
| 7 | compiled_autograd | `the_autograd_compiler` — `set_autograd_compiler` doesn't hold the mutex, module declares `Py_MOD_GIL_NOT_USED`, so callable without GIL while backward reads the pointer → use-after-free. | `Py_MOD_GIL_NOT_USED` + no mutex in setter |
| 8 | compiled_autograd | `clear_cache()` — `METH_NOARGS` on a `Py_MOD_GIL_NOT_USED` module, no lock, destroys `CacheNode` tree while backward traverses it → use-after-free. | `Py_MOD_GIL_NOT_USED` + no mutex |
| 9 | compiled_autograd | `python_verbose_logger` — borrowed reference (no `Py_INCREF`), can dangle after GC. | Missing incref |

## Significant

| # | Component | Issue |
|---|-----------|-------|
| 10 | guards.cpp | `RelationalGuard` instances shared via `shared_ptr` across cloned roots — concurrent evaluation corrupts accumulated state. |
| 11 | guards.cpp | Python-facing `check()` on non-root `GuardManager` has no locking — direct calls from Python race with guard evaluation. |
| 12 | eval_frame | `previous_eval_frame` and `active_dynamo_threads` — non-atomic, race on eval frame shim install/uninstall. |
| 13 | eval_frame | `ExtraState::invalidate` can null out `root_mgr`/`code` while another thread is mid-lookup → use of nullptr or partially-invalidated entry. |
| 14 | eval_frame | `PreserveGlobalState` saves/restores `random` module global state, clobbering other threads' random state. |
| 15 | eval_frame | `convert_frame_get_fail_callback` static local — double-init race on `std::optional<py::object>`. |
| 16 | compiled_autograd | `active_rstate` global pointer + `call_cpp_tensor_pre_hooks` callable without GIL → dereference of stale/null pointer. |
| 17 | compiled_autograd | `kActivePyCompilerInterface` — public `TORCH_API` global `unique_ptr` with no synchronization; relies on implicit protection from compiled autograd mutex in a different TU. |

## Key Architectural Observations

1. **`ExtraState` has zero synchronization.** It is attached to code objects (shared across all threads) and accessed from the eval frame hot path. Every operation — lookup, insertion, move-to-front, invalidation, strategy mutation — is unprotected. This is the single largest surface area for races.

2. **Dict watcher callbacks are the most dangerous pattern.** They fire on whatever thread mutates the dict, completely asynchronously with guard evaluation. The three process-global maps they access (`dict_version_map`, `dict_to_guard_managers`, and per-GuardManager `_dict_pointers`) are all unprotected.

3. **`Py_MOD_GIL_NOT_USED` is the root enabler for compiled_autograd races.** All five exported C methods can be called without the GIL on 3.14t, but none have adequate internal synchronization. Removing this declaration is the single highest-impact fix for compiled_autograd.

## Suggested Fix Priorities

1. **Immediate:** Add a mutex to `dict_version_map` / `dict_to_guard_managers` (or use concurrent containers). This is the most likely crash site.
2. **Immediate:** Add a per-ExtraState lock protecting the cache entry list and frame_state. Use it in lookup, insertion, invalidation, and move-to-front.
3. **Immediate:** Either remove `Py_MOD_GIL_NOT_USED` from compiled_autograd, or make `set_autograd_compiler`, `clear_cache`, and `set_verbose_logger` acquire the compiled autograd mutex.
4. **Soon:** Fix `python_verbose_logger` to hold a strong reference.
5. **Soon:** Deep-copy `RelationalGuard` instances in `clone_manager` instead of sharing via `shared_ptr`.
6. **Later:** Make `active_rstate` thread-local or pass it as an argument. Make `kActivePyCompilerInterface` thread-local or explicitly synchronized.

## Reports

- [dynamo_guards_deep.md](dynamo_guards_deep.md) — 3 SEVERE, 2 Significant, 4 Minor
- [dynamo_eval_frame_deep.md](dynamo_eval_frame_deep.md) — 4 SEVERE, 5 Significant, 5 Minor
- [dynamo_compiled_autograd_deep.md](dynamo_compiled_autograd_deep.md) — 5 SEVERE, 2 Significant, 3 Minor
