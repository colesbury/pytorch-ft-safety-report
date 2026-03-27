# Free-Threading Safety Audit: multiprocessing

**Overall Risk: None**

## Files Reviewed
- `multiprocessing/init.cpp`
- `multiprocessing/init.h`

## Findings

No free-threading issues found.

## Summary

The `multiprocessing` module exposes three simple functions:
- `_multiprocessing_init`: Imports `torch.multiprocessing` and adds a `_prctl_pr_set_pdeathsig` method.
- `_set_thread_name`: Sets the current thread's name via `c10::setThreadName`.
- `_get_thread_name`: Gets the current thread's name via `c10::getThreadName`.

The `methods` initializer_list is `static` but is a const array of `PyMethodDef` structs (initialized once, never mutated). The thread name functions operate on per-thread state. The `multiprocessing_init` function does Python imports but is called once during setup. No mutable static/global state or unsafe Python C API patterns were found.
