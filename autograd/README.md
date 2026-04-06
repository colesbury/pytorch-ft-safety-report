# Autograd Free-Threading Safety Audit

Audit of `torch/csrc/autograd/` for Python 3.14t (free-threading / nogil) data races.

## Architecture Summary

The autograd subsystem bridges Python and C++ for automatic differentiation.
Key Python-facing files include:

- **python_cpp_function.cpp** -- wraps C++ autograd `Node` objects as Python
  objects; maintains global type maps.
- **python_function.cpp** -- implements `torch.autograd.Function` (custom
  Python autograd functions); has lazy-init patterns.
- **python_variable.cpp** -- wraps `at::Tensor` as Python `THPVariable`;
  uses caching import getters.
- **python_hook.cpp** -- Python hook dispatching; already uses
  `Py_BEGIN_CRITICAL_SECTION` for compiled_args.
- **python_engine.cpp** -- Python wrapper for the autograd engine; engine
  itself has extensive mutex-based synchronization.
- **profiler_python.cpp** -- Python call profiler; had two major bugs fixed
  by #178551 and #178552; some residual issues remain.

## Issues

| Severity | File | Issue |
|---|---|---|
| Significant | python_cpp_function.cpp, function.h | [functionToPyObject TOCTOU race on Node::pyobj_](function-to-pyobject-pyobj-race.md) |

<details>
<summary>Fixed (1 issue)</summary>

| Severity | File | Issue | PR |
|---|---|---|---|
| Minor | python_function.cpp | [THPFunction_setup_context lazy-init TOCTOU](setup-context-lazy-init.md) | [#179475](https://github.com/pytorch/pytorch/pull/179475) |

</details>

## Not Reported (by design)

The following were examined and determined to be safe or not worth reporting
per the audit guidelines:

- **cpp_function_types_map / cpp_function_types_set** (python_cpp_function.cpp):
  Write-once-during-init globals. All calls to `registerCppFunction()` occur
  during module init (from `init.cpp` and generated `python_functions.cpp`),
  under the import lock. `functionToPyObject()` falls back to
  `get_default_type()` for unregistered types rather than registering lazily.

- **THPVariableClass, THPFunctionClass, THPGradientEdgeClass, ParameterClass**
  (init.cpp, python_function.cpp): Write-once-during-init globals, set
  under the import lock in `THPAutograd_initExtension`.

- **DEFINE_CACHING_PYTHON_IMPORT_GETTER** (python_variable.cpp): Uses
  `pybind11::gil_safe_call_once_and_store` on pybind 2.13+, or C++11 magic
  statics on older versions. Both are thread-safe for initialization.

- **device_to_py_class_** (python_variable.cpp): Currently only supports
  XLA registration. Write-once pattern; reads are on hot path but only
  after registration is complete.

- **python_hook.cpp**: Already uses `Py_BEGIN_CRITICAL_SECTION` for dict
  iteration in `compiled_args()`. The hook operator() methods acquire GIL.

- **python_engine.cpp**: The `_reinitialize_engine` flag is only relevant
  in fork scenarios (set in `pthread_atfork` child handler before threads
  exist). The engine itself uses mutex-based synchronization.

- **python_anomaly_mode.cpp**: All methods acquire GIL before accessing
  Python objects. No shared mutable C++ state.

- **python_saved_variable_hooks.cpp**: All methods acquire GIL. No shared
  mutable C++ state.

- **python_torch_functions_manual.cpp**: `THPVariableFunctionsModule` is a
  write-once global set during init.

- **python_nested_functions_manual.cpp**: No global mutable state.

- **python_variable_indexing.cpp**: No global mutable state; all operations
  are on per-object data.

Note: profiler_python.cpp issues are tracked in [profiler/README.md](../profiler/README.md).
