# cpp_function_types_map / cpp_function_types_set unprotected concurrent access

- **Status:** Not reported (write-once-during-init; see below)
- **Severity:** ~~Minor~~ None
- **Component:** python_cpp_function.cpp

## Shared state

Two file-scope globals:
```cpp
static std::unordered_map<std::type_index, THPObjectPtr> cpp_function_types_map;
static std::unordered_set<PyTypeObject*> cpp_function_types_set;
```

## Writers

`registerCppFunction()` inserts into both containers. All call sites are in
`init.cpp` and generated `python_functions.cpp`, both during module
initialization under the import lock. There is no lazy registration path —
`functionToPyObject()` falls back to `get_default_type()` for unknown types
rather than registering them.

## Readers

- `functionToPyObject()` does a `find()` on `cpp_function_types_map` every
  time a C++ Node needs to be wrapped as a Python object. This happens
  during backward pass, when accessing `next_functions`, or when the
  autograd graph is inspected from Python.

- `THPCppFunction_Check()` does a `find()` on `cpp_function_types_set` to
  determine whether a Python object wraps a C++ autograd function.

## Race scenario

Thread A calls `functionToPyObject()` during a backward pass (or when
inspecting `grad_fn.next_functions`), which reads `cpp_function_types_map`.
Meanwhile, Thread B triggers lazy loading of a new extension module that
calls `registerCppFunction()`, inserting into the map. The concurrent
`find()` on `std::unordered_map` while an `insert()` is in progress causes
undefined behavior -- typically a crash from following a dangling bucket
pointer after a rehash.

The same race exists for `cpp_function_types_set` via
`THPCppFunction_Check()`.

## Resolution

No fix needed. All `registerCppFunction()` calls happen during module init
under the import lock. The containers are read-only after init completes.
This is a write-once-during-init pattern, which is safe per audit guidelines.
