# Free-Threading Safety Audit: xpu

**Overall Risk: MEDIUM**

## Files Reviewed
- `xpu/Event.cpp`
- `xpu/Event.h`
- `xpu/Graph.cpp`
- `xpu/MemPool.cpp`
- `xpu/Module.cpp`
- `xpu/Module.h`
- `xpu/Stream.cpp`
- `xpu/Stream.h`
- `xpu/XPUPluggableAllocator.cpp`
- `xpu/XPUPluggableAllocator.h`
- `xpu/memory_snapshot.cpp`
- `xpu/memory_snapshot.h`

## Detailed Findings

### Issue 1 -- Global mutable `THXPEventClass` and `THXPStreamClass` (Event.cpp, Stream.cpp)

**Severity: MEDIUM**
**Location:** `Event.cpp`, line 13; `Stream.cpp`, line 11; `Event.h`, line 10; `Stream.h`, line 11

```cpp
PyObject* THXPEventClass = nullptr;   // Event.cpp
PyObject* THXPStreamClass = nullptr;  // Stream.cpp
```

These are global `PyObject*` pointers that are set during module initialization (`THXPEvent_init`, `THXPStream_init`) and then read during type checks (e.g., `THXPEvent_Check`, `THXPStream_Check`). Under free-threading, if module init races with a type check from another thread, this is a data race on a pointer read/write.

In practice, module init happens during Python import which is serialized by the import lock. After init, these pointers are only read. The risk is moderate only if code attempts to use XPU events/streams before the module is fully initialized.

**Recommendation:** These could be converted to `std::atomic<PyObject*>` for safety, though the practical risk is low given import serialization.

### Issue 2 -- `THXPModule_getCurrentStream_wrap` creates tuple without GIL concern (Module.cpp)

**Severity: LOW**
**Location:** `Module.cpp`, lines ~105-124

```cpp
PyObject* output_tuple = PyTuple_New(3);
PyTuple_SetItem(output_tuple, 0, ...);
PyTuple_SetItem(output_tuple, 1, ...);
PyTuple_SetItem(output_tuple, 2, ...);
return output_tuple;
```

`PyTuple_SetItem` on a newly created tuple that has not yet been shared with any other thread is safe. The tuple is local to this function until returned. No issue.

### Issue 3 -- `current_custom_allocator` global shared_ptr (XPUPluggableAllocator.cpp)

**Severity: MEDIUM**
**Location:** `XPUPluggableAllocator.cpp`, lines ~114-141

```cpp
std::shared_ptr<c10::xpu::XPUCachingAllocator::XPUAllocator>
    current_custom_allocator;

std::shared_ptr<...> getCurrentAllocator() {
  return current_custom_allocator;
}

void changeCurrentAllocator(const std::shared_ptr<...>& allocator) {
  ...
  current_custom_allocator = allocator;
}

void custom_raw_deleter(void* ptr) {
  current_custom_allocator->raw_delete(ptr);
}
```

`current_custom_allocator` is a global `std::shared_ptr` that is read by `getCurrentAllocator()` and `custom_raw_deleter()`, and written by `changeCurrentAllocator()`. Concurrent read/write of `std::shared_ptr` is a data race (the control block reference count operations are atomic, but the pointer itself is not). Under free-threading, if an allocator is being changed while another thread is using the allocator, this could crash.

The `changeCurrentAllocator` function checks `!c10::xpu::XPUCachingAllocator::get()->initialized()` before proceeding, which means it should only be called before the allocator is used. But `custom_raw_deleter` could be called during deallocation on any thread.

**Recommendation:** Use `std::atomic<std::shared_ptr<...>>` (C++20) or protect with a mutex.

### Issue 4 -- `device_count_` static mutable (XPUPluggableAllocator.cpp)

**Severity: LOW**
**Location:** `XPUPluggableAllocator.cpp`, line 7

```cpp
static c10::DeviceIndex device_count_ = 0;
```

Written in `XPUPluggableAllocator::init()` and read in `createCustomAllocator()`. These are called during setup, not concurrently with normal operation. Low risk.

### Issue 5 -- `XPUPluggableAllocator` internal state properly mutex-protected

**Severity: LOW**
**Location:** `XPUPluggableAllocator.cpp`, lines ~9-54

The `allocation_metadata_` map is protected by `allocator_mutex_` in both `malloc` and `raw_delete`. This is correctly synchronized.

### Issue 6 -- `PyList_GetItem` borrowed reference in `gatherForwardTraceback` (called indirectly)

**Severity: LOW**

The XPU module's `_xpu_memorySnapshot` function calls `py_symbolize` which in turn processes `CapturedTraceback` objects. The symbolization itself is covered in the profiler_python report. The XPU-specific code just collects traceback pointers and passes them to the symbolizer.

### Issue 7 -- `THXPEvent_init` / `THXPStream_init` type setup (Event.cpp, Stream.cpp)

**Severity: LOW**
**Location:** `Event.cpp`, lines ~187-200; `Stream.cpp`, lines ~182-194

These functions set `tp_base`, call `PyType_Ready`, and register the type on the module. They must be called with the GIL held (or under import lock). Under free-threading, `PyType_Ready` is safe to call concurrently for different types, but these functions also write to the global `THXPEventClass`/`THXPStreamClass` pointers (Issue 1 above).

### Issue 8 -- `_xpu_memorySnapshot` creates many Python objects (Module.cpp)

**Severity: LOW**
**Location:** `Module.cpp`, lines ~458-618

This function creates many `py::dict`, `py::list`, and `py::str` objects while holding the GIL (via pybind11). Under free-threading, all Python object creation/manipulation should use critical sections if accessing shared containers. However, all containers created here are local to the function and not shared with other threads until the function returns. Safe.

### Issue 9 -- `THXPModule_initExtension` sets module attributes (Module.cpp)

**Severity: LOW**
**Location:** `Module.cpp`, lines ~645-674

```cpp
auto set_module_attr = [&](const char* name, PyObject* v) {
  if (PyObject_SetAttrString(m, name, v) < 0) {
    throw python_error();
  }
};
```

Under free-threading, `PyObject_SetAttrString` on a module object could race with concurrent attribute reads. However, this is called during `_xpu_init` which happens during device lazy initialization, typically before XPU is used from other threads.

## Summary

The main concerns are the unsynchronized global `current_custom_allocator` shared_ptr (Issue 3) and the global `PyObject*` class pointers (Issue 1). The allocator issue is the more serious one, as `custom_raw_deleter` could be invoked from any thread during deallocation. The module-level Python type pointers are low risk given import lock serialization. The `XPUPluggableAllocator` internal state is properly mutex-protected.
