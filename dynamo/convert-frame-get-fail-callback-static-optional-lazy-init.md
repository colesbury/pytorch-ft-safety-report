# `convert_frame_get_fail_callback` static optional lazy init

- **Status:** Open
- **Severity:** Minor
- **Tier:** Tier 2
- **Component:** eval_frame
- **Source report:** [dynamo_eval_frame_v2.md](../dynamo_eval_frame_v2.md)

- **Shared state:** `convert_frame_get_fail_callback` -- a `static
  std::optional<py::object>` inside the `eval_custom` lambda in
  `dynamo__custom_eval_frame` (line 425).
- **Writer(s):** First thread to reach the `if
  (!convert_frame_get_fail_callback)` branch initializes it.
- **Reader(s):** Subsequent threads read it.
- **Race scenario:** Two threads reach the check simultaneously. Both see
  `nullopt`. Both import and assign. The `std::optional` assignment is not
  atomic, so concurrent writes could corrupt the optional's internal state.
  However, this path requires `eval_frame_override == ERROR`, which is a
  niche configuration.
- **Tier:** **Tier 2**.
- **Suggested fix:** Use `std::call_once` or `static` local variable for
  thread-safe lazy initialization.
