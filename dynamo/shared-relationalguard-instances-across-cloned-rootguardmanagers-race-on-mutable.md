# Shared `RelationalGuard` instances across cloned `RootGuardManager`s race on mutable guard state

- **Status:** Open
- **Severity:** Significant
- **Tier:** Tier 2
- **Component:** guards
- **Source report:** [dynamo_guards_v2.md](../dynamo_guards_v2.md)

- **Tier:** Tier 2
- **Shared state:** `RelationalGuard` subclass fields: `OBJECT_ALIASING::_is_first_call`/`_first_tensor`, `NO_TENSOR_ALIASING::_unique_tensors`, `SYMBOLIC_SHAPE_GUARD::_args_seen`/`_args_int`/`_args_float`, `STORAGE_OVERLAPPING` via shared `StorageOverlapChecker`
- **Writer(s):** `check_nopybind` on these guards mutates state on every call (they accumulate state across multiple invocations within one evaluation). `reset_state` clears it at the end.
- **Reader(s):** Same methods. State is both read and written during guard evaluation.
- **Race scenario:** `clone_manager` (line 3937) creates a new `RootGuardManager` that SHARES `RelationalGuard` instances via `shared_ptr` (line 3084: `cloned_mgr->_leaf_guards.emplace_back(guard)` copies the shared_ptr, not the guard). If the original root and cloned root are on different cache entries (or different code objects), they have different `_lock` mutexes. Thread A evaluates the original root (holding root A's lock), setting `SYMBOLIC_SHAPE_GUARD::_args_seen = 1`. Thread B evaluates the cloned root (holding root B's lock), also setting `_args_seen = 1`. The shared guard's mutable state is corrupted by the interleaved writes.
- **Consequence:** Incorrect guard evaluation (wrong compiled code executed), or in the case of `StorageOverlapChecker`, race on `_overlapping`/`_non_overlapping` vectors with `Py_INCREF`/`Py_DECREF` which could crash.
- **Suggested fix:** Deep-copy `RelationalGuard` instances during cloning instead of sharing them. Each `RootGuardManager` should own independent copies.
