# `_communicators` static map unprotected concurrent access

- **Status:** Open
- **Severity:** Significant
- **Tier:** Tier 2
- **Component:** nccl

- **Shared state:** `static std::unordered_map<device_list, NcclCommList, ...> _communicators` (nccl.cpp line 292).
- **Writer(s):**
  - `get_communicators` (line 295): looks up the map and may insert a new `NcclCommList` via `emplace` if no communicator exists for the given device list. Called from `broadcast`, `reduce`, `all_reduce`, `reduce_scatter`, and `all_gather` when `user_comms` is empty.
- **Reader(s):**
  - Same `get_communicators` function: calls `find` on the map.
- **Race scenario:** Thread A calls `torch.cuda.nccl.broadcast(tensors)` and Thread B calls `torch.cuda.nccl.all_reduce(tensors)` concurrently, both without providing explicit communicators. Both call `get_communicators`, which calls `_communicators.find()` and potentially `_communicators.emplace()`. Concurrent `find` + `emplace` on `std::unordered_map` is undefined behavior -- structural corruption of the hash table.
- **Note on CudaFreeMutex:** The code comment on line 291 says "accesses to this object have to be guarded by THC's CudaFreeMutex". However, `get_communicators` is called *before* the `AutoNcclGroup` guard is constructed (see e.g., `broadcast` line 588-591), and for NCCL >= 2, `AutoNcclGroup` does not even acquire `CudaFreeMutex`. The comment is stale and the map is effectively unprotected.
- **Practical risk:** This is the legacy `torch.cuda.nccl` API, not the primary ProcessGroup NCCL backend. Multi-threaded use is uncommon but not impossible. Under free-threading with Python data loaders that use NCCL for communication, concurrent calls could trigger this.
- **Consequence:** Crash (corrupted hash map), or use of stale/invalid communicator handles.
- **Suggested fix:** Add a mutex to protect `_communicators`. The mutex should be acquired in `get_communicators` itself, not rely on external callers.
