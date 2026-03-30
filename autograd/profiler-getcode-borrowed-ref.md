# Profiler getCode<> stores borrowed PyCodeObject* references

- **Status:** Open
- **Severity:** Minor
- **Tier:** Tier 2
- **Component:** profiler_python.cpp

## Shared state

The `getCode<CallType::PyModuleCall>()` and
`getCode<CallType::PyOptimizerCall>()` template specializations each store
a `PyCodeObject*` in a C++ magic static. The pointer is obtained via
`.ptr()` (a borrowed reference) from pybind11 attribute lookups:

```cpp
static auto module_call_code = []() {
    auto res = py::module::import("torch.nn")
                   .attr("Module")
                   .attr("__call__")
                   .attr("__code__")
                   .ptr();
    return (PyCodeObject*)res;
}();
```

## Writers

The `__call__` attribute of `torch.nn.Module` can be reassigned (e.g., by
monkey-patching). If it is, the `__code__` object that was retrieved may be
garbage collected, leaving the cached pointer dangling.

## Readers

`PythonTracer::recordPyCall()` compares `code.get() == module_call_code_`
on every Python call event. If the pointer is dangling, the comparison
against a freed (and potentially reused) address could give false positives
or negatives.

## Race scenario

This is less about concurrency between threads and more about lifetime
management. However, under free-threading, if one thread monkey-patches
`nn.Module.__call__` while another thread is profiling, the old code object
could be freed concurrently. The stored raw pointer would dangle. In
practice, monkey-patching `nn.Module.__call__` is rare but not unheard of
(some libraries do this for instrumentation).

## Suggested fix

Use `py::object` (which holds a strong reference) or `Py_INCREF` the code
object before storing the pointer. The C++11 magic static initialization is
fine for thread safety; the issue is purely about reference counting.
