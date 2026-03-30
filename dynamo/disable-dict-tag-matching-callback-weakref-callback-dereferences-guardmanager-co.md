# `disable_dict_tag_matching_callback` (weakref callback) dereferences `GuardManager*` concurrently with destruction

- **Status:** Open
- **Severity:** Minor
- **Tier:** Tier 1
- **Component:** guards
- **Source report:** [dynamo_guards_v2.md](../dynamo_guards_v2.md)

- **Shared state:** PyCapsule `name` field on the capsule in `_tag_safe_entries`.
- **Writer(s):** `~GuardManager()` calls `cleanup_tag_safe_entries()` which calls `PyCapsule_SetName(e.cap, "DeadGuardManager")` then `Py_CLEAR(e.wr)`.
- **Reader(s):** `disable_dict_tag_matching_callback` (line 3340) is a weakref callback that calls `PyCapsule_IsValid(self_capsule, "GuardManager*")`.
- **Race scenario:** `PyCapsule_SetName` and `PyCapsule_IsValid` on the same capsule from different threads is a data race on the `name` field (`const char*`). In practice this is unlikely to crash — both values are string literal pointers, so the read sees either the old or new pointer (no torn read on pointer-sized stores). But it is technically undefined behavior per C/C++.
- **Consequence:** Data race (UB), but unlikely to crash in practice.
- **Suggested fix:** Use an `std::atomic<bool>` flag instead of PyCapsule name mutation for the invalidation check.
