# `c_recursion_limit` non-atomic int32_t

- **Status:** Open
- **Severity:** Minor
- **Tier:** Tier 2
- **Component:** eval_frame
- **Source report:** [dynamo_eval_frame_v2.md](../dynamo_eval_frame_v2.md)

- **Shared state:** `c_recursion_limit` -- a file-scope `int32_t` (line 294
  of `eval_frame_cpp.cpp`).
- **Writer(s):** `dynamo_set_c_recursion_limit()`.
- **Reader(s):** `dynamo_get_c_recursion_limit()` and `CRecursionLimitRAII`.
- **Race scenario:** Stale read. Benign -- one frame might use a slightly
  outdated recursion limit.
- **Tier:** **Tier 2**.
- **Suggested fix:** `std::atomic<int32_t>`. Low priority.
