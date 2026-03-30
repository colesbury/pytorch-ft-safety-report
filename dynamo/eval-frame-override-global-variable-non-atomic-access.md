# `eval_frame_override` global variable non-atomic access

- **Status:** Open
- **Severity:** Significant
- **Tier:** Tier 2
- **Component:** eval_frame
- **Source report:** [dynamo_eval_frame_v2.md](../dynamo_eval_frame_v2.md)

- **Shared state:** `eval_frame_override` -- a file-scope
  `EvalFrameOverride` enum (in `eval_frame_cpp.cpp`, line 342).
- **Writer(s):** `set_eval_frame_override()` -- called from Python.
- **Reader(s):** `dynamo__custom_eval_frame` at line 443 -- reads
  `eval_frame_override` inside `eval_custom` lambda on every compiled frame
  evaluation.
- **Race scenario:** Thread A is evaluating frames and reading
  `eval_frame_override`. Thread B calls `set_eval_frame_override()` to
  change it. The read is not atomic. On x86 this is likely benign (enum is
  int-sized, loads/stores are naturally atomic), but it's technically UB in
  C++ and could be problematic on weaker architectures.
- **Tier:** **Tier 2**.
- **Suggested fix:** Make `eval_frame_override` a `std::atomic<EvalFrameOverride>`.
