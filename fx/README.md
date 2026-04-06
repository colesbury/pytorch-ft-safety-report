# FX Free-Threading Safety Audit

Deep audit of `torch/csrc/fx/` for Python 3.14t (free-threading / nogil) data races.

## Architecture Summary

The FX C extension consists of a single source file (`node.cpp`) providing
a C implementation of `torch.fx.node.Node` internals for performance:

- **`NodeBase`** — Python type (`torch._C._NodeBase`) that stores the core
  Node state: graph membership, args/kwargs, input_nodes/users dicts, and a
  sort key. Nodes form a doubly-linked list within their graph via `_prev`/`_next`
  pointers.

- **`NodeIter`** — Python iterator over the linked list of nodes.

- **`map_aggregate` / `map_arg`** — Module-level functions exposed as
  `torch._C._fx_map_aggregate` / `torch._C._fx_map_arg`. Recursively apply
  a callable over aggregate structures (tuples, lists, dicts, slices).

All `NodeBase` state is per-object (per-node, per-graph). FX graphs are
created per-compilation and not shared across threads, even under Tier 2
concurrent compilation. The only shared state is three lazy-initialized
static `PyObject*` pointers used to cache class lookups.

## Issues

| Severity | Component | Issue |
|----------|-----------|-------|
| Minor | node.cpp | [Lazy-init static `PyObject*` TOCTOU in `immutable_list_cls`, `immutable_dict_cls`, `is_node`](lazy-init-static-pyobject-toctou.md) |

## Not Reported (by design)

The following were examined and determined to be safe or not worth reporting
per the audit guidelines:

- **Per-object `NodeBase` fields** (`_prev`, `_next`, `_args`, `_kwargs`,
  `_input_nodes`, `users`, etc.): All per-node state within a per-compilation
  FX graph. Not shared across threads. The C methods (`_update_args_kwargs`,
  `_prepend`, `_remove_from_list`, `_replace_input_with`) manipulate these
  fields and the linked list without critical sections, but this is equivalent
  to the Python code they replace. FX graphs are not a concurrent data
  structure.

- **`PyDict_Next` / `PyList_GET_ITEM` borrowed references** in
  `map_aggregate` and `_update_args_kwargs`: The containers being iterated
  are node args/kwargs, which are typically `immutable_list`/`immutable_dict`
  (immutable wrappers) or tuples. These are per-graph objects, not shared
  across threads.

- **Cross-node mutations** in `_update_args_kwargs` (modifying other nodes'
  `users` dicts) and `_prepend` (relinking `_prev`/`_next` across three
  nodes): All nodes involved are in the same graph, which is not shared
  across threads.

- **`T_OBJECT_EX` member definitions** (`NodeBase_members`): Under
  free-threading, CPython's `PyMember_GetOne`/`PyMember_SetOne` use
  per-object critical sections for `T_OBJECT_EX` access from Python.
  Safe for Python-level attribute access.

- **`NodeBaseType` / `NodeIterType` static type objects**: Set during module
  init via `PyModule_AddType`. Write-once-during-init, protected by import
  lock.
