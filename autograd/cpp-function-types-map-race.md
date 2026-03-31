# cpp_function_types_map / cpp_function_types_set unprotected concurrent access

- **Status:** Open
- **Severity:** Minor
- **Tier:** Tier 1
- **Component:** python_cpp_function.cpp

## Shared state

Two file-scope globals:
```cpp
static std::unordered_map<std::type_index, THPObjectPtr> cpp_function_types_map;
static std::unordered_set<PyTypeObject*> cpp_function_types_set;
```

## Writers

`registerCppFunction()` inserts into both containers. This is called during
module initialization (e.g., from generated code registering autograd
function types) and can be called lazily when a new C++ autograd function
type is first encountered.

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

## Suggested fix

Protect both containers with a `std::mutex`, or switch to a lock-free
concurrent map. Alternatively, ensure all `registerCppFunction()` calls
happen during module init under the import lock, and document/enforce this
invariant. If registration is truly init-only, an `std::call_once` or
similar pattern that freezes the containers after init would suffice.
