# `worker_pids` static `std::map` concurrent access in DataLoader

- **Status:** Open
- **Severity:** SEVERE
- **Component:** DataLoader.cpp

- **Shared state:** `worker_pids` -- a `static std::map<int64_t, std::set<pid_t>>`
  (DataLoader.cpp:121). This is a process-global map tracking worker PIDs for
  all active DataLoader iterators.
- **Writer(s):**
  - `THPModule_setWorkerPIDs()` inserts a new entry via `worker_pids[key] = pids_set`
    (line 201). Called from Python when a new DataLoader iterator is created.
  - `THPModule_removeWorkerPIDs()` erases an entry via `worker_pids.erase(it)`
    (line 218). Called from Python when a DataLoader iterator is destroyed.
  - `THPModule_errorIfAnyWorkerFails()` calls `pid_set.clear()` (lines 152, 169)
    to clear the set of pids for a given iterator when a worker error is detected.
- **Reader(s):**
  - `THPModule_errorIfAnyWorkerFails()` iterates `worker_pids` with a range-for
    loop (line 129), reading all entries and checking worker status via `waitid`.
    Called from Python, typically from the main DataLoader thread.
- **Race scenario:** Thread A (main training thread) has one DataLoader iterator
  and calls `_error_if_any_worker_fails()`, iterating over `worker_pids`.
  Thread B (setting up a second DataLoader) calls `_set_worker_pids()` to insert
  a new entry into the map. `std::map::operator[]` may rebalance the red-black
  tree while Thread A is traversing it with iterators. This is undefined
  behavior on the map internals. Similarly, Thread A iterating while Thread C
  calls `_remove_worker_pids()` to erase an entry can invalidate iterators.
  Multiple DataLoaders in different threads is a common pattern in production.
- **Suggested fix:** Protect `worker_pids` with a `std::mutex`. All three
  operations (`set`, `remove`, `error_if_any`) are infrequent and not
  performance-sensitive, so a simple mutex is appropriate.
