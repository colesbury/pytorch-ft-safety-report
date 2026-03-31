# `python_registrations_` flat_hash_map concurrent read/write

- **Status:** Fix pending — [pytorch/pytorch#178898](https://github.com/pytorch/pytorch/pull/178898)
- **Severity:** SEVERE
- **Component:** python_dispatch
- **Source reports:** [utils__part2.md](../utils__part2.md)

- **Shared state:** `python_registrations_` -- a file-scope static
  `ska::flat_hash_map<OperatorName, ska::flat_hash_map<DispatchKey, shared_ptr<SafePyObject>>>`
  in `torch/csrc/utils/python_dispatch.cpp` (line 39-42).
- **Writer(s):**
  - `initDispatchBindings` lambda for `.impl()` (line 427):
    `python_registrations_[lib._resolve(name)].insert_or_assign(...)`.
    Called from Python when registering a custom operator implementation
    via `torch.library`.
- **Reader(s):**
  - `python_op_registration_trampoline_impl` (line 1080):
    `python_registrations_[op.operator_name()][key]`.
    Called from the dispatcher when executing a Python-registered kernel,
    which can happen on any thread that calls an op with a Python
    dispatch key.
- **Race scenario:** Thread A registers a new Python kernel via
  `torch.library`, which calls `python_registrations_[...].insert_or_assign(...)`.
  Thread B concurrently dispatches an operator that hits the Python key,
  calling `python_op_registration_trampoline_impl` which reads
  `python_registrations_[op.operator_name()][key]`. The concurrent
  read and write on `ska::flat_hash_map` (which is not thread-safe)
  causes undefined behavior -- the map's internal hash table can be in
  an inconsistent state during rehashing, leading to a crash, infinite
  loop, or corrupted iterator. Even without rehashing, concurrent
  `operator[]` (which inserts default-constructed entries on miss) racing
  with `insert_or_assign` is UB on the inner map.
- **Suggested fix:** Protect `python_registrations_` with a
  `std::shared_mutex`. Readers in `python_op_registration_trampoline_impl`
  take a shared lock; writers in the `.impl()` lambda take an exclusive
  lock. Alternatively, use a snapshot/COW approach: writers build a new
  map and atomically swap a `shared_ptr` to it.
