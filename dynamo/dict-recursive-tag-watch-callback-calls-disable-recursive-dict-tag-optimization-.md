# `dict_recursive_tag_watch_callback` races with guard evaluation on per-GuardManager state

- **Status:** Open
- **Severity:** Significant
- **Tier:** Tier 1
- **Component:** guards

## Background

Guard evaluation is a tree walk. Each `GuardManager` node corresponds
to a position in the object graph (e.g., `mod.layer.weight`). A
`RootGuardManager` owns a mutex (`_lock`) that is held for the entire
guard evaluation.

The dict-tag-matching optimization lets a "tag-safe root"
`GuardManager` skip its entire subtree by checking that the dict
version tags haven't changed. `_dict_pointers` caches per-value
`(dict_pointer, tag)` pairs for this fast path. On Python 3.12+, dict
watchers are registered so that mutations immediately disable the
optimization instead of polling tags.

## Shared state

- `GuardManager::_dict_pointers` — per-node cache mapping value
  pointers to recorded dict pointer/tag pairs.
- `GuardManager::_disable_dict_tag_matching` — bool that permanently
  disables the optimization for this node.

## Race

Two locks are involved but protect different things:

| Path | Lock held | Reads | Writes |
|------|-----------|-------|--------|
| Guard eval | `RootGuardManager::_lock` | `_disable_dict_tag_matching`, `_dict_pointers` | `_dict_pointers` (via `stash_dict_pointers`), `_disable_dict_tag_matching` |
| Dict watcher callback | `dict_to_guard_managers` | `_dict_pointers` (iterates in `unwatch_all_saved_dict_pointers`) | `_disable_dict_tag_matching` |

Neither lock covers both paths, so there are two races:

1. **`_dict_pointers`:** The eval thread inserts a new key via
   `stash_dict_pointers` while the callback iterates the map in
   `unwatch_all_saved_dict_pointers`. This is UB on
   `std::unordered_map`. It requires the GuardManager to already be
   watching dicts (from a previously recorded value), and then a new
   value triggers recording at the moment a callback fires for one of
   the old dicts.

2. **`_disable_dict_tag_matching`:** The callback sets it to `true`
   while the eval thread reads it. This is a benign stale read — the
   worst case is one extra tag check that would have been skipped.

Acquiring `RootGuardManager::_lock` in the callback would deadlock:
guard evaluation acquires `_lock` → `dict_to_guard_managers` lock, so
the reverse order in the callback is ABBA.

## Suggested fix

Have the callback only set `std::atomic<bool> _dict_tag_invalidated`.
Defer `unwatch_all_saved_dict_pointers` to the next guard evaluation,
which already holds `_lock`. The eval path checks the flag at the
start and does the cleanup if needed.
