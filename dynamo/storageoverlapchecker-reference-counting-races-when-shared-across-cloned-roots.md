# `StorageOverlapChecker` reference counting races when shared across cloned roots

- **Status:** Open
- **Severity:** Minor
- **Tier:** Tier 2
- **Component:** guards
- **Source report:** [dynamo_guards_v2.md](../dynamo_guards_v2.md)

- **Tier:** Tier 2
- **Shared state:** `StorageOverlapChecker::_overlapping` and `_non_overlapping` vectors of `PyObject*`
- **Writer(s):** `add` calls `Py_INCREF`, `reset` calls `Py_DECREF`, `_overlapping`/`_non_overlapping` vectors are mutated.
- **Reader(s):** `maybe_check` reads the vectors.
- **Race scenario:** `StorageOverlapChecker` is shared via `shared_ptr` through `STORAGE_OVERLAPPING` guards, which are `RelationalGuard`s shared across cloned roots (issue #5). If two threads evaluate guards on different roots sharing the same checker, they race on `add`/`reset`/`maybe_check`.
- **Consequence:** Corrupted vectors, incorrect reference counts.
- **Suggested fix:** Addressed by fixing issue #5 (deep-copy relational guards during cloning).
