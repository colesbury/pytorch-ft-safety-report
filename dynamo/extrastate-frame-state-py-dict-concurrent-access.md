# `ExtraState::frame_state` (py::dict) concurrent access

- **Status:** Open
- **Severity:** Significant
- **Tier:** Tier 2
- **Component:** eval_frame

## Shared state

`ExtraState::frame_state` — a `py::dict` (Python dict) attached to a
code object's `ExtraState`.

## Writers

The Python-side compilation callback receives a pointer to `frame_state`
(via `extract_frame_state`) and mutates it during compilation to record
dynamic shape information.

## Readers

Another thread compiling the same function, or the main thread accessing
frame_state while a recompilation occurs on another thread.

## Race scenario

Thread A and Thread B both trigger compilation for the same code object.
Both receive the same `frame_state` dict pointer. Both mutate it
concurrently from Python.

Note: individual dict C API operations (`PyDict_SetItem`, etc.) are
internally synchronized in CPython 3.14t via per-object critical sections.
The issue is **compound access**: the compilation logic reads shape
information from `frame_state`, makes decisions based on what it read, and
writes updated shape information back. Two threads doing this concurrently
can produce inconsistent results (e.g., both read the same initial state,
make conflicting dynamic-shape decisions, and one overwrites the other's
updates).

This is one manifestation of the broader issue that concurrent compilation
of the same code object is not designed to be thread-safe.

## Suggested fix

Protect compilation with a per-code-object lock, or serialize
compilation of the same code object.
