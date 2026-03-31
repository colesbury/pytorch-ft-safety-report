# Free-Threading Audit: c10d Python Bindings, Ops, Reducer, and SymmetricMemory

## Architecture Summary

This audit covers the c10d distributed subsystem's Python-facing components:

- **init.cpp** (~4400 lines): The main Python bindings file for `torch._C._distributed_c10d`. Binds the Reducer, Logger, Store hierarchy, ProcessGroup, SymmetricMemory, and assorted utility functions. Uses pybind11 with `py::call_guard<py::gil_scoped_release>()` on most methods.

- **Ops.cpp**: Dispatcher op implementations that route collective operations (allreduce, broadcast, etc.) to the appropriate ProcessGroup backend by device type. Stateless pass-through functions.

- **Functional.cpp**: Functional wrappers around collective ops for use with `torch.compile`. Uses `resolve_process_group` and `register_work` but contains no mutable global state itself.

- **reducer.cpp/.hpp**: The DDP Reducer. Manages gradient buckets, autograd hooks, and communication. Each Reducer instance is per-model and uses an internal `mutex_` for synchronization. The autograd hooks fire from the autograd engine thread and are serialized by the Reducer's own mutex.

- **GroupRegistry.cpp/.hpp**: Manages a global map of named ProcessGroups with a `std::shared_mutex`. Has a `thread_isolation_mode` flag that switches between a global registry and a per-thread `RankLocal` registry.

- **SymmetricMemory.cpp/.hpp**: Manages symmetric memory allocators and persistent allocations. Uses per-map mutexes (`group_info_mutex`, `persistent_alloc_mutex`, `AllocatorMap::mutex_`).

- **NVSHMEMSymmetricMemory.cpp**: NVSHMEM-specific allocator. Uses its own `mutex_` for allocations/rendezvous and a separate `rank_map_mutex` for rank mapping. Has a TOCTOU issue on NVSHMEM initialization.

- **quantization/**: Pure compute functions (float-to-bfloat16 conversion), no global mutable state.

- **python_placement.cpp/.h**: Stateless pybind11 type definitions for DTensor placement types (Shard, Replicate, Partial). No mutable state.

### Concurrency model

DDP is typically used with one Reducer per process. The Reducer's autograd hooks fire from the autograd engine thread, and most Reducer methods are called from Python with the GIL released. The Reducer's internal `mutex_` serializes these accesses.

The GroupRegistry, SymmetricMemory subsystem, and debug level are truly global and accessed across threads.

## SEVERE Issues

### 1. MemPoolAllocatorMap has no synchronization

- **Shared state:** `MemPoolAllocatorMap::mempool_allocators_` (`std::unordered_map<c10::DeviceType, std::shared_ptr<c10::Allocator>>`)
- **Writer(s):** `register_mempool_allocator()` inserts into the map, called during backend initialization
- **Reader(s):** `get_mempool_allocator()` reads from the map, called from any thread doing symmetric memory operations
- **Race scenario:** Thread A calls `get_mempool_allocator(CUDA)` to look up an allocator while thread B calls `register_mempool_allocator(CUDA, alloc)` from a different initialization path. The concurrent read and write on `std::unordered_map` causes structural corruption (rehash during lookup).
- **Severity:** SEVERE -- concurrent access to `std::unordered_map` can crash
- **Suggested fix:** Add a `std::mutex` to `MemPoolAllocatorMap`, mirroring the pattern already used in `AllocatorMap` right above it in the same file.

### 2. NVSHMEM `rank_to_global_rank_dev_map` accessed outside lock scope

- **Shared state:** `rank_to_global_rank_dev_map` (`static std::unordered_map<std::string, int*>`)
- **Writer(s):** `NVSHMEMPeerAllocInfo` constructor inserts into the map while holding `rank_map_mutex`
- **Reader(s):** `NVSHMEMSymmetricMemory::get_rank_to_global_rank()` and `get_rank_to_global_rank_dev()` acquire `rank_map_mutex`, look up in the map, then return a reference/pointer to the value. The reference remains valid only as long as no rehash occurs.
- **Race scenario:** Thread A calls `get_rank_to_global_rank()`, gets a `const vector<int>&` from the map, drops the lock, then reads from the vector. Thread B simultaneously constructs a new `NVSHMEMPeerAllocInfo` for a different group, inserting into `rank_to_global_rank_map`, causing a rehash that invalidates the reference Thread A is reading from. This corrupts the vector read or causes a use-after-free.
- **Severity:** SEVERE -- the returned reference can be invalidated by concurrent insertion
- **Suggested fix:** Return by value instead of by reference from `get_rank_to_global_rank()`, or hold the lock for the entire duration the reference is in use. Alternatively, use `std::map` which does not invalidate references on insertion.

### 3. NVSHMEM initialization TOCTOU race

- **Shared state:** `static bool is_initialized` in `initialize_nvshmem_with_store()`
- **Writer(s):** Set to `true` after NVSHMEM init completes
- **Reader(s):** Checked at function entry to skip re-initialization
- **Race scenario:** Two threads call `NVSHMEMSymmetricMemoryAllocator::alloc()` concurrently. Both see `is_initialized == false`, both proceed to call `nvshmemx_init_attr()`. Double-initializing NVSHMEM is undefined behavior and will likely crash.
- **Severity:** SEVERE -- double NVSHMEM initialization causes undefined behavior
- **Suggested fix:** Use `std::call_once` or a `std::mutex` to protect the initialization.

## Significant Issues

### 4. `thread_isolation_mode` is a non-atomic global bool used to branch on registry type

- **Shared state:** `static bool thread_isolation_mode` in GroupRegistry.cpp
- **Writer(s):** `set_thread_isolation_mode()` called from Python (via `_set_thread_isolation_mode`)
- **Reader(s):** Every call to `register_process_group()`, `resolve_process_group()`, `unregister_process_group()`, and `unregister_all_process_groups()` reads it
- **Race scenario:** Thread A calls `set_thread_isolation_mode(true)` while Thread B is in the middle of `resolve_process_group()`. Thread B reads `thread_isolation_mode == false` at the branch, enters the global `process_registry.resolve_group()` path. Meanwhile Thread A has already started registering groups in `RankLocal`. Thread B gets a stale result or a TORCH_CHECK failure because the group was registered in the wrong registry.
- **Severity:** Significant -- a torn read on the bool itself is unlikely to crash on x86, but the semantic inconsistency (groups registered in one registry, resolved from another) causes incorrect behavior
- **Suggested fix:** Make `thread_isolation_mode` a `std::atomic<bool>`.

### 5. `get_group_info` returns a reference to map entry while holding then releasing the lock

- **Shared state:** `group_info_map` (`static std::unordered_map<std::string, GroupInfo>`)
- **Writer(s):** `set_group_info()` inserts into the map under `group_info_mutex`
- **Reader(s):** `get_group_info()` acquires `group_info_mutex`, looks up the entry, then returns a mutable reference (`GroupInfo&`) after releasing the lock
- **Race scenario:** Thread A calls `get_group_info("group1")` and gets a `GroupInfo&`. Thread B calls `set_group_info("group2")`, which triggers a rehash of `group_info_map`, invalidating the reference Thread A holds. Thread A then accesses the invalidated reference, causing a use-after-free.
- **Severity:** Significant -- the returned reference can dangle after a rehash
- **Suggested fix:** Return `GroupInfo` by value, or keep the lock alive via RAII while the reference is in use. Since `set_group_info` CHECKs against duplicates, switching to `std::map` (which doesn't rehash) would also fix it but is less clean.

### 6. `g_debug_level` is a non-atomic global read/written without synchronization

- **Shared state:** `g_debug_level` (`DebugLevel` enum, in `debug.cpp`)
- **Writer(s):** `setDebugLevel()` and `setDebugLevelFromEnvironment()`, callable from Python via `set_debug_level` / `set_debug_level_from_env`
- **Reader(s):** `debug_level()`, called from Reducer constructor and throughout distributed code
- **Race scenario:** Thread A calls `set_debug_level(Detail)` while Thread B reads `debug_level()` during Reducer construction. Without free-threading, the GIL serialized these. Under free-threading, this is a data race on a non-atomic variable.
- **Severity:** Significant -- technically undefined behavior per C++ memory model. In practice on x86 the read is likely to see a consistent value (the enum is small), but this is still a standards violation.
- **Suggested fix:** Make `g_debug_level` a `std::atomic<DebugLevel>`.

## Minor Issues

### 7. `gil_checker` function pointer is a non-atomic global

- **Shared state:** `static gil_checker_t gil_checker` in `ProcessGroupNCCL.cpp`, accessed via `get_gil_checker()`
- **Writer(s):** `registerGilChecker()` in `init.cpp` sets it during static initialization
- **Reader(s):** `launchAsyncGilCheck()` and the NCCL watchdog read it
- **Race scenario:** The write happens during static initialization of `init.cpp` (before any threads could reasonably read it), and the reader is the NCCL watchdog which starts much later. In practice this is write-once-before-use.
- **Severity:** Minor -- the write happens at static init time which is serialized by the loader/import lock, so the race window is effectively zero. However, the type is a bare function pointer (not `std::atomic`), which is technically a data race if another DSO's static initializer reads it concurrently.
- **Suggested fix:** Use `std::atomic<gil_checker_t>` for correctness, or leave as-is since it is effectively write-once-during-init.

### 8. `is_finalizing_` is a non-atomic static bool in SymmetricMemory.cpp

- **Shared state:** `static bool is_finalizing_` in the anonymous namespace of SymmetricMemory.cpp
- **Writer(s):** `AllocatorMap::~AllocatorMap()` sets it to `true` during static destruction
- **Reader(s):** `is_finalizing()` called from `NVSHMEMAllocation::~NVSHMEMAllocation()` (and potentially other destructors) to avoid calling CUDA functions after driver shutdown
- **Race scenario:** During process exit, multiple destructors running on different threads could read `is_finalizing_` while `AllocatorMap`'s destructor sets it. The read is a simple bool check; a stale `false` means we attempt CUDA calls slightly too late, which is the pre-existing behavior.
- **Severity:** Minor -- worst case is a redundant CUDA call during shutdown that would fail anyway
- **Suggested fix:** Make `is_finalizing_` a `std::atomic<bool>`.

## Files With No Issues Found

- **Ops.cpp**: All functions are stateless pass-throughs to ProcessGroup backends. No global mutable state.
- **Functional.cpp**: Uses `resolve_process_group` and `register_work` (which have their own synchronization) but contains no mutable globals. The `str_to_reduce_op` map is `const`.
- **quantization/quantization.cpp, quantization.h, quantization_utils.h, quantization_gpu.h**: Pure compute kernels with no global state.
- **python_placement.cpp/.h**: Stateless pybind11 type definitions. No mutable state.
- **reducer.cpp/.hpp**: The Reducer itself is well-synchronized with its internal `mutex_`. The `RpcContext` uses `std::atomic` for its context pointer. Autograd hooks are serialized by the mutex. The Reducer is a per-model instance, not shared global state. No free-threading issues found within the Reducer's own code.
- **GroupRegistry.cpp**: The `GroupRegistry` class itself uses `std::shared_mutex` correctly. Only the `thread_isolation_mode` flag (Issue 4) and the static `process_registry` instance access pattern have issues.
- **AllocatorMap** in SymmetricMemory.cpp: Correctly uses `std::mutex` on all methods. No issues.
