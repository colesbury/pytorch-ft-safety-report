# Concurrent access to `cache_entry_list` (std::list) during lookup, insertion, invalidation, and LRU reordering

- **Status:** Open
- **Severity:** SEVERE
- **Tier:** Tier 1
- **Component:** eval_frame
- **Source report:** [dynamo_eval_frame_v2.md](../dynamo_eval_frame_v2.md)

- **Shared state:** `ExtraState::cache_entry_list` -- a `std::list<CacheEntry>`
  attached to a code object shared across all threads.
- **Writer(s):**
  - `create_cache_entry()` -- called after a cache miss during compilation
    (`emplace_front`/`emplace_back`, plus iterator bookkeeping).
  - `ExtraState::move_to_front()` / `move_to_back()` -- called from `lookup()`
    on cache hit and from `invalidate()`.
  - `ExtraState::invalidate()` -- called from Python-side dict watcher /
    mutation guard callbacks on **whatever thread triggers the mutation**
    (e.g., a data-loader thread doing a lazy import that triggers a dict
    watcher on a guarded `__dict__`). This calls `cache_entry->invalidate()`
    which mutates the `CacheEntry` fields, then calls `move_to_back()` which
    splices the list.
  - `destroy_extra_state()` -- the `_PyCode_SetExtra` free function, called
    when a code object is deallocated or when `reset_code` is called.
- **Reader(s):**
  - `lookup()` -- iterates `cache_entry_list` with a range-for loop,
    reading each `CacheEntry`'s fields (`root_mgr`, `backend`, `code`,
    `diff_guard_root_mgr`).
  - `extract_cache_entry()` -- returns `cache_entry_list.front()`.
  - `_debug_get_cache_entry_list()` -- iterates the list from Python.
  - `CacheEntry::next()` -- walks the list via `_owner_loc` iterator.
- **Race scenario:** Thread A calls a compiled function and is inside
  `lookup()`, iterating `cache_entry_list` and evaluating guards on each
  `CacheEntry`. Thread B (a data-loader thread, or another compile thread)
  triggers a dict watcher callback that calls `ExtraState::invalidate()`,
  which mutates a `CacheEntry`'s fields (setting `code` to `None`,
  `root_mgr` to `nullptr`) and splices the list via `move_to_back()`.
  Thread A is now iterating a structurally modified `std::list` -- the
  iterator may be invalidated or point to moved nodes, and the `CacheEntry`
  fields Thread A is reading (`root_mgr`, `code`) are being concurrently
  overwritten. This is undefined behavior on the `std::list` internals and
  on the `CacheEntry` fields. Likely crash: null dereference of `root_mgr`
  in `run_root_guard_manager`, or use of a dangling iterator.
- **Tier:** **Tier 1** for the invalidation path (dict watcher callbacks fire
  on data-loader threads). **Tier 2** for concurrent lookup + create.
- **Suggested fix:** Protect `ExtraState` with a mutex. The critical sections
  are: `lookup()` (read lock), `create_cache_entry()` (write lock),
  `move_to_front()`/`move_to_back()` (write lock), `invalidate()` (write
  lock). A `std::shared_mutex` (reader-writer lock) on `ExtraState` would
  allow concurrent lookups while serializing mutations. Alternatively, use
  atomic operations on the list structure (e.g., RCU-style read-copy-update).
