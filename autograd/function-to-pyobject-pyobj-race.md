# functionToPyObject TOCTOU race on Node::pyobj_

- **Status:** Open
- **Severity:** Significant
- **Component:** python_cpp_function.cpp, function.h

## Shared state

`Node::pyobj_` -- a raw `PyObject*` member on the autograd `Node` class,
accessed via `pyobj()` and `set_pyobj()` with no synchronization. It acts
as a weak reference from a C++ Node to its Python wrapper.

## Writers

- `functionToPyObject()` sets `pyobj_` via `cdata->set_pyobj(obj.release())`
  when creating a new Python wrapper for a C++ Node (line 315).
- `THPCppFunction_clear()` sets `pyobj_` to `nullptr` (line 118) when the
  Python wrapper is being destroyed.

## Readers

- `functionToPyObject()` reads `pyobj_` (line 296) to check if a wrapper
  already exists, and reads it again (line 318) to return the result.
- `THPCppFunction_traverse()` and other Python GC machinery may read it.

## Race scenario

Thread A calls `functionToPyObject(node)`. It reads `cdata->pyobj()` on
line 296 and sees `nullptr`. It proceeds to allocate a new wrapper and set
`pyobj_`. Meanwhile Thread B also calls `functionToPyObject(node)` for the
same Node. It also reads `nullptr` from `pyobj()` (TOCTOU), allocates a
second wrapper, and sets `pyobj_` -- overwriting Thread A's wrapper. The
wrapper created by Thread A now has a dangling weak-reference, and refcount
is wrong.

Additionally, Thread A reading `pyobj_` while Thread B calls
`THPCppFunction_clear()` and sets it to `nullptr` is a read-write race on
a non-atomic pointer.

## Suggested fix

Make `pyobj_` an `std::atomic<PyObject*>` and use `compare_exchange` in
`functionToPyObject()` to atomically install the wrapper. If the CAS fails,
discard the newly created wrapper and use the existing one.
