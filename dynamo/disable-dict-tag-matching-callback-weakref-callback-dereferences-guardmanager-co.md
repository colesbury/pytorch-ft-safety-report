# `disable_dict_tag_matching_callback` (weakref callback) dereferences `GuardManager*` concurrently with destruction

- **Status:** Open
- **Severity:** SEVERE
- **Tier:** Tier 1
- **Component:** guards
- **Source report:** [dynamo_guards_v2.md](../dynamo_guards_v2.md)

- **Tier:** Tier 1
- **Shared state:** `GuardManager*` stored inside a `PyCapsule` in `_tag_safe_entries`, and the `GuardManager` object itself.
- **Writer(s):** `~GuardManager()` calls `cleanup_tag_safe_entries()` (line 2957) which iterates `_tag_safe_entries` and calls `PyCapsule_SetName(e.cap, "DeadGuardManager")` then `Py_CLEAR(e.wr)`.
- **Reader(s):** `disable_dict_tag_matching_callback` (line 3340) is registered as a weakref callback. When the weakly-referenced Python object dies, this callback fires on whatever thread triggers GC. It calls `PyCapsule_IsValid(self_capsule, "GuardManager*")` then dereferences the pointer.
- **Race scenario:** Thread A destroys a `GuardManager` -- the destructor enters `cleanup_tag_safe_entries` and has invalidated entries [0..k-1] but not yet entry [k]. Thread B triggers GC on the Python object referenced by entry [k]'s weakref, firing `disable_dict_tag_matching_callback`. The callback sees a valid capsule (name still "GuardManager*") and dereferences the pointer. But the `GuardManager` destructor is actively running on Thread A, and the object may be partially destroyed. Under free-threading, weakref callbacks fire without GIL serialization.
- **Consequence:** Use-after-free or accessing partially-destroyed object.
- **Suggested fix:** Use an `std::atomic<bool>` in a shared control block (or inside the GuardManager) that both the destructor and the callback check/set atomically, instead of relying on `PyCapsule` name as a synchronization mechanism. Or ensure the weakref callback and the destructor are ordered (e.g., by dropping the weakrefs first, which should fire callbacks synchronously in CPython before the destructor proceeds).
