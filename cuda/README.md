# CUDA Free-Threading Issues

Issues in `torch/csrc/cuda/` affecting free-threaded Python 3.14t.

## Architecture Summary

The `torch/csrc/cuda/` directory contains the Python-facing CUDA module bindings,
the pluggable allocator infrastructure, memory snapshot tooling, and the legacy
NCCL wrapper (`torch.cuda.nccl`). Most of these files are thin wrappers around
C10 CUDA APIs and are called from Python with standard GIL semantics (or explicit
`gil_scoped_release`). The main concurrency concerns are:

1. **CUDAPluggableAllocator**: A global `shared_ptr` (`current_custom_allocator`)
   is read by the deleter function on any thread that deallocates a CUDA tensor,
   and written by `changeCurrentAllocator`.

2. **nccl.cpp**: A static `_communicators` map is read/written by `get_communicators`
   without any mutex protection (despite a stale comment claiming otherwise).

3. **Module.cpp**: The `cudaLockMutex`/`cudaUnlockMutex` pattern relied on the GIL
   for deadlock prevention, which is broken under free-threading. Being removed
   (no callers) in [#178833](https://github.com/pytorch/pytorch/pull/178833).

The `memory_snapshot.cpp` `CallbackManager` is properly protected by its own mutex.
The `shared/` directory files, `Event.cpp`, `python_nccl.cpp`, `python_comm.cpp`,
and `GdsFile.cpp` contain no concerning shared mutable state -- they operate on
per-call locals or use proper synchronization.

## Issues

| Severity | Component | Issue |
|----------|-----------|-------|
| Significant | nccl | [`_communicators` static map unprotected concurrent access](nccl-communicators-map-unprotected-concurrent-access.md) |
| Minor | CUDAPluggableAllocator | [`current_custom_allocator` shared_ptr unsynchronized read/write](current-custom-allocator-shared-ptr-unsynchronized-read-write.md) |

<details>
<summary>Fixed (1 issue)</summary>

| Severity | Component | Issue | PR |
|----------|-----------|-------|----|
| Minor | Module | [`cudaMutexGILState` implicit GIL-based deadlock prevention broken](cuda-mutex-gil-state-implicit-locking-broken.md) | [#178833](https://github.com/pytorch/pytorch/pull/178833) |

</details>
