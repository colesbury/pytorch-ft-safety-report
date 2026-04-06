# `run_const_fold` inactive buffer path races with model state

- **Status:** Open
- **Severity:** SEVERE
- **Component:** aoti_runtime/model_container.h

- **Shared state:** Model instance's `constants_map_` and `constants_`
  (`shared_ptr` members on `AOTInductorModelBase`)
- **Writer:** `run_const_fold(inactive_buffer=true)` temporarily swaps a model's
  constant pointers to the inactive buffer, runs constant folding, then swaps
  back — holding `model_exec_mutex_` in **shared** mode only.
- **Race scenario:** Thread A is in `run_const_fold`, swaps model's constants to
  inactive buffer. Thread B calls `swap_constant_buffer()` (unique lock), flips
  `use_secondary_`. Thread A's "swap back" reads `use_secondary_` (which has
  changed), and the model ends up pointing at the wrong buffer.
- **Consequence:** Model reads wrong constants, potential use-after-free if the
  buffer is subsequently freed.
- **Suggested fix:** Hold unique lock for the entirety of `run_const_fold`, or
  use a separate lock that `swap_constant_buffer()` also acquires.
