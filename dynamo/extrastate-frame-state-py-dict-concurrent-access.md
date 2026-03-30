# `ExtraState::frame_state` (py::dict) concurrent access

- **Status:** Open
- **Severity:** Significant
- **Tier:** Tier 2
- **Component:** eval_frame
- **Source report:** [dynamo_eval_frame_v2.md](../dynamo_eval_frame_v2.md)

- **Shared state:** `ExtraState::frame_state` -- a `py::dict` (Python dict).
- **Writer(s):** The Python-side compilation callback receives a pointer to
  `frame_state` (via `extract_frame_state`) and mutates it during
  compilation to record dynamic shape information.
- **Reader(s):** Another thread compiling the same function, or the main
  thread accessing frame_state while a recompilation occurs on another thread.
- **Race scenario:** Thread A and Thread B both trigger compilation for the
  same code object. Both receive the same `frame_state` dict pointer. Both
  mutate it concurrently from Python. Under free-threading, concurrent
  mutations to the same Python dict without holding the per-object lock can
  cause corruption (Python 3.14t has per-object locks for dicts, but only
  for Python-level access via bytecode -- C-level access via `PyDict_SetItem`
  from multiple threads without the GIL is still racy unless using the
  critical section API).
- **Tier:** **Tier 2**.
- **Suggested fix:** Protect compilation with a per-code-object lock, or
  ensure `frame_state` access goes through Python critical sections.
