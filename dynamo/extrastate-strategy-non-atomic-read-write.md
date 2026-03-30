# `ExtraState::strategy` non-atomic read/write

- **Status:** Open
- **Severity:** Significant
- **Tier:** Tier 2
- **Component:** eval_frame
- **Source report:** [dynamo_eval_frame_v2.md](../dynamo_eval_frame_v2.md)

- **Shared state:** `ExtraState::strategy` -- a `FrameExecStrategy` struct
  containing two `FrameAction` enum values.
- **Writer(s):** `extra_state_set_exec_strategy()` -- called from
  `dynamo__custom_eval_frame` (line 649) after compilation, and from
  `dynamo_set_code_exec_strategy` / `dynamo_skip_code_recursive`.
- **Reader(s):** `extra_state_get_exec_strategy()` -- called from
  `dynamo__custom_eval_frame` (line 509) at the start of every frame
  evaluation.
- **Race scenario:** Thread A is evaluating a frame and reads `strategy`.
  Thread B just finished compiling the same function and writes a new
  `strategy` (e.g., setting `cur_action = SKIP`). The struct write is not
  atomic (it's two enum fields), so Thread A could read a torn value where
  `cur_action` is updated but `recursive_action` is stale, or vice versa.
  This could cause incorrect frame action decisions (e.g., skipping a frame
  that should be compiled, or vice versa).
- **Tier:** **Tier 2**.
- **Suggested fix:** Protect with the same `ExtraState` mutex from issue #1,
  or pack the two enum values into a single `std::atomic<uint32_t>`.
