# Model container management APIs bypass `model_exec_mutex_`

- **Status:** Open
- **Severity:** SEVERE
- **Component:** aoti_runtime/model_container.h

`AOTInductorModelContainer` uses `model_exec_mutex_` (shared_mutex) to
synchronize `run()` (shared mode) with `swap_constant_buffer()` (unique mode).
However, `update_constant_buffer()`, `extract_constants_map()`, and
`free_inactive_constant_buffer()` operate on the same shared mutable state
**without holding any lock**. From Python, these are currently serialized by the
GIL — all pybind methods in `aoti_runner/pybind.cpp` are exposed without
`py::call_guard<py::gil_scoped_release>()`. Under free-threading, this implicit
serialization disappears.

## `update_constant_buffer` races with `run()`

- **Shared state:** `constants_map_` (an `unordered_map`), `constants_array_`,
  `constant_folded_`, `use_secondary_`
- **Writer:** `update_constant_buffer()` calls `insert_or_assign` on the
  `ConstantMap` and writes `constant_folded_` — no lock held.
- **Reader:** `run()` holds `model_exec_mutex_` in shared mode; models read
  `constants_map_` and `constants_array_` through `shared_ptr` copies.
- **Race scenario:** Thread A calls `update_constant_buffer(use_inactive=false)`,
  modifying the active `ConstantMap`. Thread B is in `run()` iterating the same
  map. Concurrent `insert_or_assign` + iteration on `unordered_map` is UB —
  bucket chain corruption, crash.

## `extract_constants_map` races with `update_constant_buffer`

- **Shared state:** `constants_map_`, `use_secondary_`
- **Reader:** `extract_constants_map()` iterates the map via `find()` — no lock.
- **Race scenario:** Thread A iterates the map in `extract_constants_map`.
  Thread B concurrently inserts via `update_constant_buffer`. Rehash triggered
  by insert invalidates Thread A's iterator.

## `free_inactive_constant_buffer` races with `swap_constant_buffer`

- **Shared state:** `constant_blob_`, `constant_blob_secondary_`,
  `use_secondary_`
- **Writer:** `free_inactive_constant_buffer()` reads `use_secondary_` to select
  the inactive buffer, then resets it — no lock held.
- **Writer:** `swap_constant_buffer()` flips `use_secondary_` under unique lock.
- **Race scenario:** Thread A calls `free_inactive_constant_buffer()`, reads
  `use_secondary_ == false`, decides to free secondary. Thread B calls
  `swap_constant_buffer()`, flips `use_secondary_` to `true` (secondary is now
  active). Thread A frees the now-active buffer — use-after-free during
  inference.

## Suggested fix

Extend `model_exec_mutex_` to cover all public entry points. At minimum,
`update_constant_buffer()` and `free_inactive_constant_buffer()` should acquire
unique mode; `extract_constants_map()` should acquire shared mode.
