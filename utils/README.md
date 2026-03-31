# torch/csrc/utils/ Free-Threading Issues

Issues in `torch/csrc/utils/` affecting free-threaded Python 3.14t.

## Issues

| Severity | Component | Issue |
|----------|-----------|-------|
| SEVERE | python_dispatch | [`python_registrations_` flat_hash_map concurrent read/write](python-registrations-flat-hash-map-concurrent-access.md) ⏳ [#178898](https://github.com/pytorch/pytorch/pull/178898) |
| SEVERE | python_dispatch | [`leaked_python_filenames_` vector concurrent push_back invalidates pointers](leaked-python-filenames-vector-concurrent-access.md) ⏳ [#178898](https://github.com/pytorch/pytorch/pull/178898) |
| SEVERE | tensor_types | [`options_from_string` lazy-init maps with broken guard pattern](options-from-string-lazy-init-maps-broken-guard-pattern.md) |
| Significant | device_lazy_init | [`device_lazy_init` TOCTOU race on `is_initialized` arrays](device-lazy-init-toctou-on-is-initialized.md) ⏳ [#178911](https://github.com/pytorch/pytorch/pull/178911) |
| Significant | tensor_new | [`static std::string sig` data race in `_validate_sparse_compressed_tensor_args_template`](validate-sparse-compressed-static-string-race.md) |
| Minor | device_lazy_init | [`is_in_bad_fork` and `at_fork_registered` non-atomic bool arrays](is-in-bad-fork-at-fork-registered-non-atomic-bool-arrays.md) ⏳ [#178911](https://github.com/pytorch/pytorch/pull/178911) |

## Not reported (safe patterns)

The following were reviewed and determined to be safe:

- **`disabled_torch_function` / `disabled_torch_dispatch`**
  (disable_torch_function.cpp): Write-once-during-init globals, set under
  the import lock. Read-only after init.

- **`static PythonArgParser` instances** (tensor_new.cpp, python_arg_parser.cpp): `PythonArgParser` is immutable after construction. `raw_parse()` and `check_deprecated()` only read `signatures_`. The `ParsedArgs` buffer is per-call stack-local. C++11 magic statics guarantee thread-safe one-time construction. Safe for concurrent use.

- **`type_map` and `numpy_compatibility_arg_names`** (python_arg_parser.cpp): File-scope statics with constant initialization, read-only after program start.

- **`should_allow_numbers_as_tensors`** (python_arg_parser.cpp): Function-local `static std::unordered_set` with constant initialization. Read-only after init.

- **`parseKind` / `parseAliasAnalysisKind`** (python_dispatch.cpp): Function-local static maps with constant initialization. Read-only after init.

- **`python_symnode.cpp` statics** (`get_symint_class`, etc.): Uses `py::gil_safe_call_once_and_store` on pybind11 2.13+, which is explicitly thread-safe. The `#else` fallback uses C++11 magic statics. Safe.

- **`PyObjectPreservation`** (pyobject_preservation.cpp): Uses `compare_exchange_strong`, `compare_exchange_weak`, and explicit memory ordering. Designed for thread safety.

- **`structseq.cpp`**: No global mutable state. `returned_structseq_repr` operates on local data only.

- **`init.cpp`**: Throughput benchmark bindings. No shared mutable state in the binding code.

- **`invalid_arguments.cpp`**: `format_invalid_args` operates on function-local data. The `PyDict_Next` on kwargs is already wrapped in `Py_BEGIN_CRITICAL_SECTION`. Tuple items via `PyTuple_GET_ITEM` are safe (tuples are immutable).

- **`parse_privateuseone_backend`** (tensor_types.cpp): Function-local static strings, safe via C++11 magic statics (read-only after init).

- **`PythonTorchFunctionTLS` state** (disable_torch_function.cpp): The `DisableTorchFunctionSubclass` and `DisableTorchFunction` context managers operate on thread-local state (`PythonTorchFunctionTLS`), not globals. Safe.
