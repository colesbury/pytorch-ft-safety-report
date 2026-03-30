# `active_dynamo_threads` counter race

- **Status:** Open
- **Severity:** Significant
- **Tier:** Tier 2
- **Component:** eval_frame
- **Source report:** [dynamo_eval_frame_v2.md](../dynamo_eval_frame_v2.md)

- **Shared state:** `ModuleState::active_dynamo_threads` -- an `int` in
  module state.
- **Writer(s):** `increment_working_threads()` and
  `decrement_working_threads()` -- read-increment/decrement-write without
  any synchronization.
- **Race scenario:** Thread A and Thread B both call
  `increment_working_threads`. Both read the same value (e.g., 0), both
  write 1. The counter is now 1 instead of 2. Later, Thread A calls
  `decrement_working_threads`, counter goes to 0, and
  `enable_eval_frame_default()` disables the shim while Thread B still
  expects it.
- **Tier:** **Tier 2**.
- **Suggested fix:** Use `std::atomic<int>` or protect with the same mutex
  as issue #9.
