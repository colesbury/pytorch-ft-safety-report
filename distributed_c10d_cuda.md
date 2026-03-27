# Free-Threading Safety Audit: distributed_c10d_cuda

## Overall Risk: LOW

## Files Analyzed
- `distributed/c10d/cuda/CUDAEventCache.cpp`
- `distributed/c10d/cuda/StreamBlock.cpp`
- `distributed/c10d/cuda/utils.cpp`

## Detailed Analysis

### 1. CUDAEventCache.cpp - `get()` with thread_local static
- **Location**: Lines 43-56
- **Issue**: `CUDAEventCache::get()` uses a `static thread_local std::map` to maintain per-thread, per-device caches. The `thread_local` keyword guarantees each thread has its own copy, so there is no cross-thread sharing of the map itself.
- **Risk**: NONE - `thread_local` provides per-thread isolation by design.

### 2. CUDAEventCache.cpp - `create()` with shared deleter
- **Location**: Lines 13-41
- **Issue**: The `create()` method returns a `shared_ptr<CUDAEvent>` with a custom deleter that captures `cache = shared_from_this()`. The deleter acquires `cacheMutex_` before pushing the event back. The `cacheMutex_` also protects the `eventsArray_` deque in `create()`.
- **Risk**: NONE - The deleter may be called from any thread (when the last `shared_ptr` reference is dropped), but the mutex correctly serializes access to `eventsArray_`.

### 3. StreamBlock.cpp - Registry-based factory
- **Location**: Lines 1-14
- **Issue**: `C10_DEFINE_REGISTRY(StreamBlockRegistry, StreamBlock, std::chrono::milliseconds)` defines a global registry. `block_stream()` calls `StreamBlockRegistry()->Create("CUDA", timeout)`. The registry is populated at static init time via `C10_REGISTER_CLASS`.
- **Risk**: NONE - Reads from an immutable registry after static initialization.

### 4. utils.cpp - `deviceSupportsMulticast()`
- **Location**: Lines 15-36
- **Issue**: Pure function that queries CUDA driver API. No mutable state. `c10::cuda::DriverAPI::get()` returns a singleton that is initialized once.
- **Risk**: NONE

## Summary

This group has no free-threading safety issues. The `CUDAEventCache` uses `thread_local` storage for per-thread isolation and a `std::mutex` for the shared event pool. The other files are either stateless or use properly synchronized registries.
