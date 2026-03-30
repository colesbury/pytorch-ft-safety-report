# `breakpoint_code_objects` (std::unordered_set) concurrent access

- **Status:** Open
- **Severity:** SEVERE
- **Tier:** Tier 2
- **Component:** eval_frame
- **Source report:** [dynamo_eval_frame_v2.md](../dynamo_eval_frame_v2.md)

- **Shared state:** `breakpoint_code_objects` -- a file-scope
  `std::unordered_set<PyCodeObject*>`.
- **Writer(s):** `register_breakpoint_code()` -- called from Python to add
  code objects.
- **Reader(s):** `dynamo__custom_eval_frame` at line 463 -- calls
  `breakpoint_code_objects.count(cached_code)` during eval_custom.
- **Race scenario:** Thread A is evaluating a frame in `eval_custom` and
  calls `breakpoint_code_objects.count()`. Thread B simultaneously calls
  `register_breakpoint_code()` which calls `insert()` on the same
  `unordered_set`. Concurrent read + write on `std::unordered_set` is
  undefined behavior (potential rehash during read).
- **Tier:** **Tier 2** (requires concurrent compiled function execution +
  breakpoint registration, though breakpoint registration is relatively rare).
- **Suggested fix:** Use a `std::mutex` around accesses, or use a concurrent
  set. Since this is a debugging feature, a simple mutex is appropriate.
