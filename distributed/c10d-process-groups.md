# Free-Threading Audit: c10d Process Groups

## Architecture Summary

The `c10d` process group layer provides the C++ backend for `torch.distributed`.
The main classes are:

- **`ProcessGroup`** (`ProcessGroup.hpp`/`.cpp`): Base class that dispatches
  collective operations to backend-specific implementations. Holds maps from
  device types to backends (`deviceTypeToBackend_`, `backendTypeToBackend_`).
  Also defines a `WorkRegistry` for tracking in-flight work objects.

- **`ProcessGroupNCCL`**: NCCL backend. Manages NCCL communicators in
  `devNCCLCommMap_` (protected by `mutex_`), a watchdog thread, a heartbeat
  monitor thread, and work metadata lists. Has significant shared mutable
  state both per-instance and global (e.g. `ncclCommMemPoolMap`).

- **`ProcessGroupGloo`**: Gloo backend. Uses a thread pool for async work,
  with a work queue protected by `workMutex_`.

- **`ProcessGroupWrapper`**: Debug wrapper that dispatches to an underlying
  backend after performing collective consistency checks via a Gloo backend.

- **`ProcessGroupMPI`**: MPI backend. Uses a worker thread with a mutex-protected
  queue.

- **`ProcessGroupUCC`**: UCC backend. Uses a static mutex-protected `Comm`
  singleton.

**Concurrency model for free-threading:** Under free-threading, multiple Python
threads can invoke collectives concurrently on the same `ProcessGroup` instance.
The distributed layer already has extensive internal threading (watchdog,
heartbeat monitor, Gloo worker threads, on-completion hook thread), so much
of the internal state is already protected by mutexes. The main issues are
shared mutable state that was implicitly protected by the GIL (calls from
Python into ProcessGroup methods) rather than explicit locks.

Note: The distributed c10d code is somewhat different from the dynamo/compile
use-case described in the audit guidelines. There are no dict watcher callbacks
or ExtraState. The relevant free-threading concern is: can multiple Python
threads call ProcessGroup methods concurrently and hit unprotected shared state?

## SEVERE Issues

### 1. `ProcessGroup::barrier()` races on static `at::Tensor`

- **Status:** Fix pending — [pytorch/pytorch#178896](https://github.com/pytorch/pytorch/pull/178896)
- **Shared state:** `static at::Tensor tensor` in `ProcessGroup::barrier()`
  (`ProcessGroup.hpp:796`). This is a function-local static that is
  **reassigned on every call** -- it is not a write-once variable.
- **Writer(s):** Every call to `ProcessGroup::barrier()` assigns to this static
  tensor (lines 801-817), choosing a device-appropriate empty tensor.
- **Reader(s):** The same `tensor` is passed to `op.call()` on line 829 and
  potentially registered as work on line 836.
- **Race scenario:** Thread A calls `barrier()` and sets `tensor` to a CUDA
  tensor. Before Thread A reaches `op.call()`, Thread B calls `barrier()` and
  overwrites `tensor` with a CPU tensor. Thread A now passes the wrong tensor
  to the NCCL barrier, causing a device mismatch crash or use-after-free if
  the original tensor was deallocated during the reassignment.
- **Severity:** SEVERE -- concurrent writes to a non-atomic `at::Tensor`
  (which involves refcount manipulation and pointer updates) cause data
  corruption and likely segfault.
- **Suggested fix:** Make `tensor` a local variable instead of static. There
  is no reason for it to be static; it was likely done to avoid repeated
  allocations, but `at::empty({1}, ...)` is cheap.

### 2. `ProcessGroupNCCL::groupRanks()` and `ProcessGroupGloo::groupRanks()` race on static vector

- **Shared state:** `static std::vector<uint64_t> globalRanks(size_)` in
  `ProcessGroupNCCL::groupRanks()` (`ProcessGroupNCCL.cpp:2686`) and
  `ProcessGroupGloo::groupRanks()` (`ProcessGroupGloo.cpp:728`).
- **Writer(s):** Every call that takes the `if` branch calls `std::iota` on
  the static vector, writing `0, 1, 2, ...` into it. C++11 magic statics
  guarantee the initialization is thread-safe, but `std::iota` runs on every
  subsequent call, not just the first.
- **Reader(s):** The caller receives a const reference to the static vector
  and reads its contents. Other threads may be simultaneously calling
  `std::iota` on it.
- **Race scenario:** Thread A calls `groupRanks()`, gets a reference to the
  static vector, and begins iterating. Thread B calls `groupRanks()` and
  `std::iota` writes to the vector's elements while Thread A reads them.
  Since `std::vector` element writes are not atomic, this is undefined
  behavior. In practice, since the values written are always the same
  (0, 1, 2, ...), the corruption is unlikely to manifest, but it is still
  a data race per the C++ standard.
- **Severity:** SEVERE per the strict definition (concurrent read/write to a
  non-thread-safe container), though practically low-impact since the same
  values are always written.
- **Suggested fix:** Initialize the vector contents during the magic static
  initialization itself using a lambda, so `std::iota` only runs once:
  ```cpp
  static std::vector<uint64_t> globalRanks = [&] {
    std::vector<uint64_t> v(size_);
    std::iota(v.begin(), v.end(), 0);
    return v;
  }();
  ```

## Significant Issues

### 3. `ProcessGroupNCCL::globalRank()` returns reference to magic static initialized from instance state

- **Shared state:** `static int globalRank = rank_` in
  `ProcessGroupNCCL::globalRank()` (`ProcessGroupNCCL.cpp:2676`).
- **Writer(s):** C++11 magic static initialization runs exactly once, on the
  first call. It captures `rank_` from whichever `ProcessGroupNCCL` instance
  calls it first.
- **Reader(s):** All subsequent calls from any `ProcessGroupNCCL` instance
  return this same value.
- **Race scenario:** Two `ProcessGroupNCCL` instances with different ranks
  call `globalRank()` concurrently during initialization. C++11 guarantees
  only one will initialize the static, but the "winner" is nondeterministic.
  After initialization, all PG instances share the same `globalRank` value,
  which may be wrong for PG instances created later.
- **Severity:** Significant -- this is a correctness issue rather than a crash.
  The wrong global rank will be used for debug logging, dump file naming, and
  the `dumpPipe` path. Under the GIL, the first PG created was deterministic
  (always the default PG), so this worked. Under free-threading, if PGs are
  created on different threads, the order is no longer deterministic.
- **Suggested fix:** Store `globalRank` as an instance member set during
  construction, or use a proper global set once during default PG creation.

### 4. `ProcessGroup::getBackend(DeviceType)` reads and writes `deviceTypeToBackend_` without synchronization

- **Shared state:** `deviceTypeToBackend_` (`ProcessGroup.hpp:1020-1021`),
  an `unordered_map` member of `ProcessGroup`.
- **Writer(s):** `getBackend(DeviceType)` at `ProcessGroup.cpp:88` inserts
  into `deviceTypeToBackend_` as a cache fill. `setBackend()` also writes to
  it (`ProcessGroup.hpp:850`). `release_resources()` clears it
  (`ProcessGroup.cpp:153`).
- **Reader(s):** `getBackend(DeviceType)` reads from it at line 72-73.
  Many other methods iterate over it: `setGroupName`, `setGroupDesc`,
  `enableCollectivesTiming`, `setTimeout`, `shutdown`, `abort`.
- **Race scenario:** Thread A calls a collective (which calls `getBackend`).
  Thread B calls `release_resources()` or `setBackend()`. Since
  `deviceTypeToBackend_` is an `unordered_map` with no lock, concurrent
  read and write (or write and write) is undefined behavior.
- **Severity:** Significant -- in typical usage, backends are set up during
  init and `release_resources` is called during teardown, so the window is
  narrow. But if a thread is still running collectives while another thread
  destroys the PG, this will crash.
- **Suggested fix:** Either document that `ProcessGroup` methods must not be
  called concurrently with `setBackend`/`release_resources`, or add a
  `shared_mutex` to protect the backend maps.

### 5. `ProcessGroupNCCL::isInitialized()` reads `devNCCLCommMap_` without holding `mutex_`

- **Shared state:** `devNCCLCommMap_` (`ProcessGroupNCCL.hpp:1349`).
- **Writer(s):** `initNCCLComm()` inserts into the map (under `mutex_`).
  `destroyNCCLComms()` erases from it (under `mutex_`).
- **Reader(s):** `isInitialized()` at line 1163 calls `.empty()` on
  `devNCCLCommMap_` *without* the lock, then acquires the lock for iteration
  at line 1166.
- **Race scenario:** Thread A calls `isInitialized()`. Between the `.empty()`
  check and the lock acquisition, Thread B inserts a communicator. Thread A
  could observe an inconsistent state. More critically, calling `.empty()` on
  an `unordered_map` that is concurrently being modified is undefined behavior.
- **Severity:** Significant -- the TOCTOU between line 1163 and 1166 could
  cause `isInitialized()` to return `false` when it should return `true`.
  The `.empty()` call itself could crash if the map is being rehashed.
- **Suggested fix:** Move the `.empty()` check inside the lock.

### 6. `ProcessGroupNCCL::getCommSplitCounter()` iterates `devNCCLCommMap_` without lock

- **Shared state:** `devNCCLCommMap_`.
- **Writer(s):** `initNCCLComm()`, `destroyNCCLComms()`, etc. (under `mutex_`).
- **Reader(s):** `getCommSplitCounter()` (`ProcessGroupNCCL.cpp:3362`) iterates
  `devNCCLCommMap_` without acquiring `mutex_`.
- **Race scenario:** Thread A calls `getCommSplitCounter()` while Thread B is
  inserting a new communicator. Iterating an `unordered_map` during concurrent
  modification is undefined behavior.
- **Severity:** Significant -- will crash if a rehash occurs during iteration.
- **Suggested fix:** Acquire `mutex_` before iterating.

### 7. `ProcessGroupNCCL::shutdown()` iterates `devNCCLCommMap_` outside lock

- **Shared state:** `devNCCLCommMap_`.
- **Writer(s):** Various methods that add/remove communicators (under `mutex_`).
- **Reader(s):** `shutdown()` at line 1600 iterates `devNCCLCommMap_` to call
  `waitReady()` after releasing the lock at line 1597.
- **Race scenario:** If another thread is concurrently modifying
  `devNCCLCommMap_` (e.g., a late `initNCCLComm` call), the iteration at
  line 1600 races with the map modification.
- **Severity:** Significant -- concurrent iteration and modification of
  `unordered_map` is undefined behavior. In practice, shutdown is typically
  serialized, but under free-threading a late collective could trigger
  `initNCCLComm` while shutdown is in progress.
- **Suggested fix:** Copy the communicator shared_ptrs under the lock, then
  iterate the copy outside the lock.

## Minor Issues

### 8. `ProcessGroupNCCL::abortComms()` reads `devNCCLCommMap_` without `mutex_` for `ncclCommMemPoolMap` cleanup

- **Shared state:** `devNCCLCommMap_`.
- **Writer(s):** `initNCCLComm()` etc.
- **Reader(s):** `abortComms()` at line 1518 iterates `devNCCLCommMap_` while
  holding `ncclCommMemPoolMapMutex` but **not** `mutex_`. The lock for
  `mutex_` is only acquired afterward at line 1523.
- **Race scenario:** Thread A calls `abortComms()`. While iterating
  `devNCCLCommMap_` under `ncclCommMemPoolMapMutex`, Thread B inserts a new
  communicator via `initNCCLComm()` (which holds `mutex_` but not
  `ncclCommMemPoolMapMutex`), causing iterator invalidation.
- **Severity:** Minor -- `abortComms` is called during error handling, which
  is a relatively rare path and typically all collective work has stopped.
- **Suggested fix:** Acquire `mutex_` before iterating `devNCCLCommMap_`
  (may need to nest locks carefully to avoid deadlock with
  `ncclCommMemPoolMapMutex`), or copy the keys under the lock first.

### 9. Non-atomic counters `seqCollective_`, `seqP2P_`, `op_id_` in ProcessGroupNCCL

- **Shared state:** `seqCollective_`, `seqP2P_`, `op_id_` are plain
  `uint64_t` members of `ProcessGroupNCCL`.
- **Writer(s):** Incremented in `collective()` (`ProcessGroupNCCL.cpp:3782-3784`),
  `pointToPoint()`, and `collectiveCoalesced()`.
- **Reader(s):** Read in various places for work metadata, logging, and
  flight recorder entries.
- **Race scenario:** Two threads call collectives concurrently on the same
  ProcessGroupNCCL. Both increment `op_id_++` without synchronization, leading
  to lost increments or torn reads.
- **Severity:** Minor -- the sequence numbers are used for logging and debugging.
  Lost or duplicated sequence numbers cause confusing logs but not crashes.
  Under the GIL, collectives on the same PG were serialized.
- **Suggested fix:** Make these `std::atomic<uint64_t>`.

### 10. Non-atomic `collectiveCounter_` in ProcessGroupGloo

- **Shared state:** `collectiveCounter_` (`ProcessGroupGloo.hpp:464`), a
  `uint32_t` member.
- **Writer(s):** `nextTag()` (`ProcessGroupGloo.cpp:685`) increments it.
- **Reader(s):** Same method returns the pre-increment value, used as a tag
  for gloo operations.
- **Race scenario:** Two threads call collectives concurrently, both calling
  `nextTag()`. The non-atomic increment could produce duplicate tags.
- **Severity:** Minor -- duplicate tags would cause gloo to match operations
  incorrectly, but ProcessGroupGloo documents that collectives must be called
  in the same order across processes. Multi-threaded concurrent collective
  calls on the same Gloo PG is not a supported use case.
- **Suggested fix:** Make `collectiveCounter_` `std::atomic<uint32_t>` if
  concurrent usage is desired.

### 11. Non-atomic `coalescing_state_` and related fields in ProcessGroupNCCL

- **Shared state:** `coalescing_state_`, `coalescedDevice_`,
  `coalescedComm_`, `coalescedAsync_` in `ProcessGroupNCCL`.
- **Writer(s):** `startCoalescing()`, `endCoalescing()`, and `collective()`.
- **Reader(s):** `collective()` reads these in the hot path.
- **Race scenario:** Thread A calls `startCoalescing()` followed by collectives,
  while Thread B also uses coalescing on the same PG. The interleaved
  writes to `coalescing_state_` cause incorrect coalescing behavior.
- **Severity:** Minor -- coalescing is a single-thread operation by design.
  Under the GIL, concurrent coalescing on the same PG was not possible.
- **Suggested fix:** Document that coalescing is single-threaded, or add a
  per-thread coalescing state.

### 12. `ProcessGroupNCCL::ncclCommCounter_` non-atomic increment

- **Shared state:** `ncclCommCounter_` (`ProcessGroupNCCL.hpp:1318`).
- **Writer(s):** Incremented during `initNCCLComm()` to generate unique
  scope keys.
- **Reader(s):** Read in the same function to compose store keys.
- **Race scenario:** Two threads trigger communicator creation concurrently,
  both reading and incrementing `ncclCommCounter_` without the lock.
- **Severity:** Minor -- `initNCCLComm` does acquire `mutex_` for map
  operations, but the counter increment may happen outside the lock. Duplicate
  keys would cause store conflicts.
- **Suggested fix:** Make atomic, or ensure all accesses are under `mutex_`.
