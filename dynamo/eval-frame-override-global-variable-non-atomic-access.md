# `eval_frame_override` global variable non-atomic access

- **Status:** Open
- **Severity:** Significant
- **Tier:** Tier 2
- **Component:** eval_frame

## Shared state

`eval_frame_override` — a file-scope `EvalFrameOverride` enum
(eval_frame_cpp.cpp line 340).

## Writers

`set_eval_frame_override()` (line 346) — called from Python in a
context-manager pattern:

- `_TorchCompileInductorWrapper._torchdynamo_orig_callable` (eval_frame.py
  line 977) sets it when `fullgraph=True`
- Restored on exit (eval_frame.py line 1066)

## Readers

`dynamo__custom_eval_frame` reads it inside the `eval_custom` lambda
(eval_frame_cpp.cpp line 440) on every compiled frame evaluation.

## Race scenario

This is per-compilation context-manager state stored in a global, not a
configuration flag. Under Tier 2, thread A enters fullgraph compilation
and sets `eval_frame_override = ERROR`. Thread B is evaluating a different
compiled function and reads `eval_frame_override` — it sees thread A's
ERROR override and raises an error on a frame that should have been
allowed, or vice versa.

The issue is not just atomicity but **semantic correctness**: a global is
being used for what should be per-thread state.

## Suggested fix

Make `eval_frame_override` `thread_local`, or replace with a Python
context variable.
