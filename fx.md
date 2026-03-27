# Free-Threading Safety Audit: fx

**Overall Risk: High**

## Files Reviewed
- `fx/node.cpp`
- `fx/node.h`

## Findings

### 1. Lazy-Initialized Static `PyObject*` Without Synchronization
**Severity: High**
**Location:** `fx/node.cpp`, lines 30-54

```cpp
inline static PyObject* immutable_list_cls() {
  static PyObject* immutable_list_cls = nullptr;
  if (!immutable_list_cls) {
    immutable_list_cls =
        import_from("torch.fx.immutable_collections", "immutable_list");
  }
  return immutable_list_cls;
}

inline static PyObject* immutable_dict_cls() {
  static PyObject* immutable_dict_cls = nullptr;
  if (!immutable_dict_cls) {
    immutable_dict_cls =
        import_from("torch.fx.immutable_collections", "immutable_dict");
  }
  return immutable_dict_cls;
}

inline static bool is_node(PyObject* obj) {
  static PyObject* node_cls = nullptr;
  if (!node_cls) {
    node_cls = import_from("torch.fx.node", "Node");
  }
  return PyObject_TypeCheck(obj, reinterpret_cast<PyTypeObject*>(node_cls));
}
```

Three functions (`immutable_list_cls`, `immutable_dict_cls`, `is_node`) use lazy initialization of static `PyObject*` pointers with a check-then-set pattern that has no synchronization. Under free-threading:
- Two threads could race to set the static pointer, potentially leaking a reference (the loser's `import_from` result is never decref'd).
- A thread could read a partially-written pointer value.
- The `import_from` call itself does Python C API work (PyImport_ImportModule, PyObject_GetAttrString) that under nogil may execute concurrently with other Python operations.

These should use `Py_BEGIN_CRITICAL_SECTION` / `Py_END_CRITICAL_SECTION` or `std::call_once` with appropriate Python API safety, or be initialized eagerly during module init.

### 2. Borrowed References from Python Containers Without Critical Section
**Severity: Medium**
**Location:** `fx/node.cpp`, multiple locations in `map_aggregate` and `NodeBase__update_args_kwargs`

The `map_aggregate` function (lines 61-148) uses `PyTuple_GET_ITEM`, `PyList_GET_ITEM`, and `PyDict_Next` which return borrowed references. Under free-threading, if another thread modifies the container concurrently, these borrowed references could become dangling. Key locations:

- Line 74: `PyTuple_GET_ITEM(a, i)` -- borrowed ref from tuple
- Line 98: `PyList_GET_ITEM(a, i)` -- borrowed ref from list
- Line 117: `PyDict_Next(a, &pos, &key, &value)` -- borrowed refs from dict

In `NodeBase__update_args_kwargs` (line 324):
```cpp
while (PyDict_Next(input_nodes, &pos, &key, &value)) {
    PyDict_DelItem(reinterpret_cast<NodeBase*>(key)->users, self);
}
```
This iterates over `input_nodes` with borrowed references while potentially modifying `users` dicts on other nodes. Under free-threading, another thread could modify `input_nodes` or `key->users` concurrently.

### 3. Direct Python Object Mutation Without Critical Section
**Severity: Medium**
**Location:** `fx/node.cpp`, `NodeBase` methods

The `NodeBase` struct stores Python object pointers (`graph`, `name`, `op`, `target`, etc.) as raw `PyObject*` members exposed via `PyMemberDef`. Under free-threading, concurrent access to these members (e.g., reading `_args` while another thread calls `_update_args_kwargs` which does `Py_CLEAR(node->_args)`) can cause use-after-free.

The linked-list operations in `set_prev`, `set_next`, `remove_from_list`, and `_prepend` (lines 177-451) modify prev/next pointers with Py_INCREF/Py_DECREF sequences that are not atomic and not protected by critical sections.

## Summary

The `fx/node.cpp` file has significant free-threading risks. The lazy-initialized static PyObject* pointers are the highest priority issue as they affect all uses of `map_aggregate`, `is_node`, etc. The borrowed reference and direct mutation issues are also concerning but are somewhat mitigated if FX graph manipulation is typically single-threaded. All three categories of issues should be addressed for safe operation under free-threading.
