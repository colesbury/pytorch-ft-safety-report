# `is_skip_guard_eval_unsafe` non-atomic bool

- **Status:** Open
- **Severity:** Minor
- **Tier:** Tier 2
- **Component:** eval_frame
- **Source report:** [dynamo_eval_frame_v2.md](../dynamo_eval_frame_v2.md)

- **Shared state:** `is_skip_guard_eval_unsafe` -- a file-scope `bool`
  (declared `extern` in `eval_frame.h`, defined in `eval_frame.c` line 728).
- **Writer(s):** `set_skip_guard_eval_unsafe()` -- from Python.
- **Reader(s):** `dynamo__custom_eval_frame` (line 543, 583) and `lookup()`
  (line 146 parameter) -- on every frame evaluation.
- **Race scenario:** Stale read. Thread A reads the old value while Thread B
  sets a new one. This could cause one extra frame to use the wrong guard
  evaluation mode, which is relatively benign (either an unnecessary full
  guard eval or one frame with skip-guard-eval when it shouldn't).
- **Tier:** **Tier 2**.
- **Suggested fix:** Make it `std::atomic<bool>`. Low priority since a stale
  read is benign.
