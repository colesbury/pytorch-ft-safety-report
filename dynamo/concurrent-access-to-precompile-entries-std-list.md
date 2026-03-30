# Concurrent access to `precompile_entries` (std::list)

- **Status:** Open
- **Severity:** SEVERE
- **Tier:** Tier 2
- **Component:** eval_frame
- **Source report:** [dynamo_eval_frame_v2.md](../dynamo_eval_frame_v2.md)

- **Shared state:** `ExtraState::precompile_entries` -- a
  `std::list<PrecompileEntry>`.
- **Writer(s):**
  - `_load_precompile_entry()` -- called from Python to add entries.
  - `_reset_precompile_entries()` -- called from Python to clear entries.
- **Reader(s):**
  - `lookup()` -- iterates `precompile_entries` in its range-for loop,
    calling `run_root_guard_manager` on each entry.
- **Race scenario:** Thread A is inside `lookup()` iterating
  `precompile_entries`. Thread B calls `_reset_precompile_entries()` or
  `_load_precompile_entry()`, structurally modifying the list. Thread A's
  iterator is invalidated. Crash from corrupted list internals.
- **Tier:** **Tier 2** (requires concurrent compiled function execution and
  precompile entry manipulation).
- **Suggested fix:** Same mutex on `ExtraState` as issue #1.
