# `use_secondary_` read without synchronization in management APIs

- **Status:** Open
- **Severity:** Significant
- **Component:** aoti_runtime/model_container.h

- **Shared state:** `use_secondary_` (plain `bool`) in
  `AOTInductorModelContainer`
- **Writer:** `swap_constant_buffer()` flips it under unique lock.
- **Reader:** `update_constant_buffer()`, `extract_constants_map()`,
  `get_constant_blob_ptr()`, `get_constants_map()`, `get_constants_array()`,
  `free_inactive_constant_buffer()` all read it **without any lock**.
- **Race scenario:** Thread A calls `update_constant_buffer(use_inactive=true)`,
  reads `use_secondary_ == false`, decides to update secondary. Thread B calls
  `swap_constant_buffer()`, flips `use_secondary_` to `true`. Thread A's update
  goes to the **now-active** buffer instead of the inactive one.
- **Consequence:** Wrong buffer mutated — active buffer corrupted during
  inference, or inactive buffer left stale.
- **Suggested fix:** Make `use_secondary_` an `std::atomic<bool>`, or ensure all
  readers hold at least a shared lock on `model_exec_mutex_`.
