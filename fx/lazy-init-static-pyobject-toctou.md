# Lazy-init static `PyObject*` TOCTOU in `immutable_list_cls`, `immutable_dict_cls`, `is_node`

- **Status:** Open
- **Severity:** Minor
- **Component:** node.cpp

## Shared state

Three function-local static `PyObject*` pointers, all using the same pattern:

```cpp
inline static PyObject* immutable_list_cls() {
  static PyObject* immutable_list_cls = nullptr;
  if (!immutable_list_cls) {
    immutable_list_cls =
        import_from("torch.fx.immutable_collections", "immutable_list");
  }
  return immutable_list_cls;
}
```

The same pattern appears in `immutable_dict_cls()` (line 39) and `is_node()`
(line 48).

Note: these are NOT C++11 magic statics. The `nullptr` initialization is a
constant init (no thread-safety concern), but the subsequent check-then-write
is a manual TOCTOU.

## Writers

First call to each function triggers `import_from()`, which calls
`PyImport_ImportModule` + `PyObject_GetAttrString`, then stores the result.

## Readers

- `immutable_list_cls()` and `immutable_dict_cls()` — called from
  `map_aggregate()`, which is called from `_update_args_kwargs`,
  `_replace_input_with`, `_fx_map_aggregate`, and `_fx_map_arg`.
- `is_node()` — called from `map_aggregate` (via `map_arg`), `_prepend`,
  `__lt__`, `__gt__`, etc.

These are all called during FX graph manipulation, typically during
`torch.compile()`.

## Race scenario

Thread A and Thread B both call `torch.compile()` for the first time
simultaneously (Tier 2). Both enter `immutable_list_cls()`, both see
`nullptr`, both call `import_from()`. Both get the same `PyObject*` (same
class from the same module — `PyImport_ImportModule` serializes via the
import lock). Both write the pointer. The second write leaks the first
thread's reference (one `PyObject*` leaked per function, at most three
total).

The pointer write is technically a C++ data race (undefined behavior), though
on all practical architectures aligned pointer writes are atomic and both
threads write the same value.

## Suggested fix

Use C++11 magic statics:
```cpp
inline static PyObject* immutable_list_cls() {
  static PyObject* cls =
      import_from("torch.fx.immutable_collections", "immutable_list");
  return cls;
}
```

This leverages the C++ standard's thread-safe initialization guarantee.
The same fix applies to `immutable_dict_cls()` and `is_node()`.

Alternatively, use `pybind11::gil_safe_call_once_and_store` for consistency
with other PyTorch code.
