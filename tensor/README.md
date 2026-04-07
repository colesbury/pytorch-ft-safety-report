# Tensor Free-Threading Safety Audit

Deep audit of `torch/csrc/tensor/` for Python 3.14t (free-threading / nogil) data races.

## Architecture Summary

The tensor module consists of a single source file (`python_tensor.cpp`) that:

- Defines **`PyTensorType`** — a `PyTypeObject` subclass representing legacy
  tensor types like `torch.FloatTensor`, `torch.cuda.DoubleTensor`, etc.

- Manages a **metaclass** (`torch.tensortype`) that provides `__instancecheck__`,
  `dtype`, `layout`, `is_cuda`, `is_xpu`, `is_sparse`, `is_sparse_csr` on the
  type objects themselves.

- Tracks the **default backend** via a file-scoped static `Backend default_backend`,
  exposed through `get_default_dispatch_key()` and `get_default_device()`.

- Tracks the **default dtype** via globals in `c10/core/DefaultDtype.cpp`
  (`default_dtype`, `default_dtype_as_scalartype`, `default_complex_dtype`),
  exposed through `get_default_scalar_type()`.

### Lifecycle

All `PyTensorType` objects and the `tensor_types` vector are created during
`initialize_python_bindings()`, which runs during `torch` module init
(serialized by the import lock). After init, these are never modified — all
fields are read-only.

The only post-init mutable state is `default_backend` and the three default
dtype globals in `DefaultDtype.cpp`, which are written by
`torch.set_default_tensor_type()` (deprecated) and `torch.set_default_dtype()`.

## Issues

No SEVERE, Significant, or Minor issues found.

## Not Reported (by design)

The following were examined and determined to be safe or not worth reporting
per the audit guidelines:

- **`default_backend` static (line 61)** — Non-atomic `Backend` enum written by
  `set_default_tensor_type()` (called from the deprecated
  `torch.set_default_tensor_type()` and from `torch.set_default_dtype()`), read
  by `get_default_dispatch_key()` and `get_default_device()` on hot paths
  (tensor constructors, argument parsing). `Backend` is an enum fitting in a
  single naturally-aligned machine word — no torn reads on any architecture. A
  stale read means one tensor is created on the old default backend, which is
  benign. This is the **configuration flag** pattern described in the audit
  guidelines. The broad audit flagged this as medium severity; it is downgraded
  here after tracing all readers and writers.

- **Default dtype globals in `c10/core/DefaultDtype.cpp`** —
  `set_default_dtype()` performs a non-atomic compound write to three globals
  (`default_dtype`, `default_dtype_as_scalartype`, `default_complex_dtype`).
  Readers may see a partially-updated state (e.g., new scalar type but old
  complex type). Each individual global is small enough for atomic hardware
  read/write (uint16\_t or equivalent). A stale or partially-updated read means
  one tensor gets the old default dtype — benign. Same configuration flag
  pattern.

- **`tensor_types` vector (line 304)** — Populated during
  `initialize_aten_types()` (module init) and never modified afterward. Read by
  `PyTensorType_Check()` (called from `py_set_default_tensor_type()`). Safe:
  write-once-during-init, protected by import lock.

- **`PyTensorType` objects** — Heap-allocated during `initialize_aten_types()`.
  All fields (`backend`, `scalar_type`, `layout`, `dtype`, `is_cuda`, `is_xpu`,
  `name`) set during init, never modified. Read-only after init.

- **`metaclass` and `tensor_type_prototype` statics** — Initialized during
  `py_initialize_metaclass()` / `initialize_python_bindings()`. Write-once-
  during-init.

- **`TORCH_WARN_ONCE` (lines 79, 436)** — Uses C++11 magic static for the
  once-flag. Thread-safe initialization guaranteed by the standard. The
  underlying `warn_always` flag in `c10/util/Exception.cpp` is a non-atomic
  bool, but a stale read only affects whether a deprecation warning is shown.

- **`set_default_storage_type` (line 306)** — Calls `PyObject_SetAttrString` to
  set `torch.Storage`. Under free-threading, CPython module dict operations are
  internally synchronized. Concurrent calls to `set_default_tensor_type` would
  race on which value wins, but this is a user programming error (concurrent
  default-setting), not a library bug.
