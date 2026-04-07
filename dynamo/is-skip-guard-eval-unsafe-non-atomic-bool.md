# `is_skip_guard_eval_unsafe` non-atomic bool

- **Status:** Open
- **Severity:** Significant
- **Tier:** Tier 2
- **Component:** eval_frame

## Shared state

`is_skip_guard_eval_unsafe` — a file-scope `bool` (declared `extern` in
`eval_frame.h` line 34, defined in `eval_frame.c` line 728).

## Writers

`set_skip_guard_eval_unsafe()` — called from Python in a context-manager
pattern:

- `_OptimizeContext.__enter__` (eval_frame.py line 798) sets it based on
  the current stance
- `_OptimizeContext.__exit__` (eval_frame.py line 811) restores the
  previous value
- Same pattern in `_TorchCompileInductorWrapper._torchdynamo_orig_callable`
  (eval_frame.py lines 1019, 1071)

## Readers

`dynamo__custom_eval_frame` (eval_frame_cpp.cpp line 540) passes the value
to `lookup()`. Also checked directly at line 581 to error on cache miss
during skip-guard mode.

## Race scenario

This is **not** a configuration flag — it is per-compilation context-manager
state stored in a global. Under Tier 2, thread A enters
`_OptimizeContext.__enter__` and sets `is_skip_guard_eval_unsafe = true`.
Thread B is evaluating a frame and reads the global — it sees thread A's
value and skips full guard evaluation when it should not, or vice versa.

The issue is not just atomicity (stale read) but **semantic correctness**:
a global is being used for what should be per-thread state. Making it
`std::atomic<bool>` would fix the UB but not the cross-thread leakage.

## Suggested fix

Make `is_skip_guard_eval_unsafe` `thread_local`, or replace with a Python
context variable.
