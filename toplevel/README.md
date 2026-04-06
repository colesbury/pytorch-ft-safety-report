# Top-Level `torch/csrc/` Free-Threading Issues

Issues in the top-level files of `torch/csrc/` affecting free-threaded Python 3.14t.

## Architecture Summary

The top-level files in `torch/csrc/` implement Python bindings for core PyTorch
types (Storage, Generator, Stream, Event, Device, Dtype) and the main module
initialization (`Module.cpp`). `DataLoader.cpp` provides signal handling and
worker process tracking for multiprocessing data loading.
`python_dimname.cpp` maintains a global cache mapping interned Python strings
to `at::Dimname` values.

Most of these files follow the pattern of setting up global state during module
init (write-once globals like `THPGeneratorClass`, `THPStorageClass`, dtype/layout
registries) and then providing Python method implementations that operate on
per-object state. The main thread-safety concerns are the few pieces of truly
shared mutable state that are accessed post-init from multiple threads.

Notable: `Storage.cpp` has already been updated for free-threading, using
`PyObjectPreservation::init_once` and `PyUnstable_EnableTryIncRef` in
`THPStorage_Wrap`. `Generator.cpp` has not received the same treatment.

## Files Reviewed

- Module.cpp, DataLoader.cpp, python_dimname.cpp, Generator.cpp
- Storage.cpp, StorageMethods.cpp, StorageSharing.cpp
- Stream.cpp, DynamicTypes.cpp, Device.cpp, Dtype.cpp, Event.cpp
- utils.cpp, serialization.cpp

## Issues

| Severity | Component | Issue |
|----------|-----------|-------|
| SEVERE | python_dimname.cpp | [`InternedStringsTable` concurrent hash map access](interned-strings-table-concurrent-access.md) |
| SEVERE | DataLoader.cpp | [`worker_pids` static `std::map` concurrent access](worker-pids-static-map-concurrent-access.md) |
| Significant | Generator.cpp | [`THPGenerator_Wrap` TOCTOU on `pyobj()` / `set_pyobj()`](thpgenerator-wrap-toctou-on-pyobj.md) |

<details>
<summary>Fixed (1 issue)</summary>

| Severity | Component | Issue | PR |
|----------|-----------|-------|----|
| SEVERE | Module.cpp | [`LogAPIUsageOnceFromPython` static `std::unordered_set` concurrent access](log-api-usage-once-static-unordered-set.md) | [#178867](https://github.com/pytorch/pytorch/pull/178867) |

</details>

## Not Reported (safe or write-once-during-init)

The following patterns were reviewed and determined to be safe or not worth
reporting:

- **Global `PyObject*` type pointers** (`THPGeneratorClass`, `THPStorageClass`,
  `THPStreamClass`, `THPEventClass`, `THPUpperModuleOfDevice`): All set during
  module init, then read-only. Protected by Python's import lock.

- **`dtype_registry` / `layout_registry`** (DynamicTypes.cpp): Fixed-size arrays
  written during init by `registerDtypeObject`/`registerLayoutObject`, then
  read-only.

- **`THPStorage_Wrap`** (Storage.cpp): Already uses `PyObjectPreservation::init_once`
  with atomic compare-and-exchange for thread-safe wrapper creation.

- **Stream.cpp `context` field**: Per-object (not global) mutable state stored as
  a Python list. Python's per-object locking under free-threading protects
  concurrent list operations.

- **`THPModule_initNames` / `THPModule_addDocStr`** (Module.cpp): Both are only
  called during module initialization, protected by Python's import lock.

- **Config flag getters/setters** (Module.cpp): Functions like
  `setAllowTF32CuDNN`, `setBenchmarkCuDNN`, etc. delegate to
  `at::globalContext()` which manages its own thread safety. Stale reads of
  config flags are benign.

- **`backCompatBroadcastWarn` / `backCompatKeepdimWarn`** (utils.cpp): Non-atomic
  bools, but these are config flags where stale reads are benign.

- **serialization.cpp**: All functions operate on local state (file descriptors,
  buffers). No shared mutable state.

- **Device.cpp, Dtype.cpp, Event.cpp**: No shared mutable state beyond
  write-once-during-init globals.
