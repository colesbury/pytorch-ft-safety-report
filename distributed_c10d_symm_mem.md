# Free-Threading Safety Audit: distributed_c10d_symm_mem

## Overall Risk: MEDIUM

## Files Analyzed
- `distributed/c10d/symm_mem/CUDASymmetricMemoryUtils.cpp`
- `distributed/c10d/symm_mem/CudaDMAConnectivity.cpp`
- `distributed/c10d/symm_mem/DMAConnectivity.cpp`
- `distributed/c10d/symm_mem/NVSHMEMSymmetricMemory.cpp`
- `distributed/c10d/symm_mem/SymmetricMemory.cpp`
- `distributed/c10d/symm_mem/cuda_mem_pool.cpp`
- `distributed/c10d/symm_mem/intra_node_comm.cpp`

## Detailed Analysis

### 1. SymmetricMemory.cpp - AllocatorMap singleton
- **Location**: Lines 29-122
- **Issue**: `AllocatorMap` is a Meyers' singleton with an internal `std::mutex` protecting `map_`, `avail_map_`, and `in_use_`. All public methods acquire the lock.
- **Risk**: NONE - Properly synchronized.

### 2. SymmetricMemory.cpp - `is_finalizing_` global bool
- **Location**: Line 21
- **Issue**: `static bool is_finalizing_ = false` is written in `AllocatorMap::~AllocatorMap()` and read in `is_finalizing()`. No synchronization. However, finalization is inherently a single-threaded process (happens during interpreter shutdown).
- **Risk**: LOW - Finalization-only flag; concurrent reads during finalization are benign (always transitioning from false to true).

### 3. SymmetricMemory.cpp - `configured_signal_pad_size_` atomic
- **Location**: Line 26
- **Issue**: `static std::atomic<size_t>` with proper acquire/release ordering in `get_signal_pad_size()` and `set_signal_pad_size()`.
- **Risk**: NONE - Properly atomic.

### 4. SymmetricMemory.cpp - `group_info_map` with mutex
- **Location**: Lines 124-125
- **Issue**: `static std::unordered_map<std::string, GroupInfo> group_info_map` protected by `static std::mutex group_info_mutex`. Properly synchronized.
- **Risk**: NONE

### 5. SymmetricMemory.cpp - `alloc_id_to_dev_ptr` and `alloc_id_to_storage` with mutex
- **Location**: Lines 128-131
- **Issue**: Two static maps protected by `static std::mutex persistent_alloc_mutex`. Properly synchronized.
- **Risk**: NONE

### 6. DMAConnectivity.cpp - DetectorMap singleton without synchronization
- **Location**: Lines 14-69
- **Issue**: `DetectorMap` is a Meyers' singleton, but it has **no internal synchronization**. The `register_detector()` method writes to `detector_map_` without locks, and `detect()` reads from `detector_map_` and writes to `cached_` without locks.
- **Risk**: MEDIUM - `register_detector()` is called from static initialization (`RegisterDetector` in `CudaDMAConnectivity.cpp`, line 126-132), which is single-threaded. `detect()` is called at runtime and could potentially be called from multiple threads concurrently. The `cached_` map acts as a lazy cache without synchronization, so concurrent calls to `detect()` for the same key could race on `cached_` insertion.
- **Mitigation**: Add a `std::mutex` to `DetectorMap` to protect `detect()`.

### 7. CudaDMAConnectivity.cpp - `RegisterDetector` static
- **Location**: Lines 125-132
- **Issue**: `static RegisterDetector register_detector_` executes `register_dma_connectivity_detector()` at static init time. Thread-safe (single-threaded static init).
- **Risk**: NONE

### 8. NVSHMEMSymmetricMemory.cpp - static maps with mutex
- **Location**: Lines 58-62
- **Issue**: `static std::mutex rank_map_mutex` and `static std::unordered_map` for rank-to-global-rank mappings. The mutex protects the maps.
- **Risk**: NONE - Properly synchronized (assuming the mutex is always acquired before access, which needs verification in the full file).

### 9. NVSHMEMSymmetricMemory.cpp - `storeExchange` static
- **Location**: Line 31
- **Issue**: `static StoreExchange storeExchange = StoreExchange("NVSHMEMSymmetricMemory")` is initialized at static init time. If `StoreExchange` is stateless or internally synchronized, this is fine.
- **Risk**: LOW

### 10. NVSHMEMSymmetricMemory.cpp - `exchanged_n_times` counter
- **Location**: Line 77 (inside `NVSHMEMPeerAllocInfo` constructor)
- **Issue**: `static int exchanged_n_times = 0` is incremented inside a constructor without synchronization. This is used only for logging.
- **Risk**: LOW - Race on a logging counter; worst case is slightly inaccurate log output.

### 11. cuda_mem_pool.cpp - static allocator caching
- **Location**: Lines 8-9
- **Issue**: `static auto allocator = get_allocator(c10::DeviceType::CUDA)` inside `cuda_symm_alloc` and `cuda_symm_free`. These are Meyers' singletons (thread-safe init). The allocator itself is a shared intrusive_ptr.
- **Risk**: NONE

### 12. cuda_mem_pool.cpp - `RegisterCUDAMemPoolAllocator` static
- **Location**: Lines 22-31
- **Issue**: Static init registration. Thread-safe.
- **Risk**: NONE

### 13. CUDASymmetricMemoryUtils.cpp - `getSymmMemBackendCUDA()` static
- **Location**: Lines 32-49
- **Issue**: `static auto val = c10::utils::get_env("TORCH_SYMMMEM")` is a Meyers' singleton. Thread-safe.
- **Risk**: NONE

### 14. intra_node_comm.cpp - static mutable globals
- **Location**: Lines 14-20
- **Issue**: `static std::vector<std::string> ENABLE_INTRA_NODE_COMM` and `TEST_INTRA_NODE_COMM` are effectively const after initialization. `static int intraNodeCommIdx = 0` is a mutable counter.
- **Risk**: LOW - `intraNodeCommIdx` is only incremented by `IntraNodeComm::rendezvous()` which creates intra-node communicators. This is typically called during initialization, not during concurrent training.

### 15. intra_node_comm.cpp - ROCm `amdsmi` lazy init
- **Location**: Lines 55-89
- **Issue**: `static void* amdsmi_handle`, `static AmdsmiApi amdsmi`, `static bool amdsmi_resolved` are lazily initialized in `getNvlMesh()` without synchronization. If two threads call `getNvlMesh()` concurrently before `amdsmi_resolved` is set, they could both attempt `dlopen()`.
- **Risk**: MEDIUM - Double `dlopen()` is benign (returns same handle), but the writes to `amdsmi_handle` and `amdsmi` struct fields could tear. Should use `std::call_once`.
- **Mitigation**: Wrap the lazy init in `std::call_once` or use Meyers' singleton.

## Summary

The symmetric memory subsystem is mostly well-synchronized with mutexes protecting shared state. The notable concerns are:

1. **`DetectorMap` in `DMAConnectivity.cpp`** lacks synchronization for its `detect()` method, which performs lazy caching. Concurrent calls could corrupt the `cached_` map.

2. **ROCm `amdsmi` lazy init in `intra_node_comm.cpp`** uses a check-then-act pattern on static variables without synchronization.

3. **`exchanged_n_times`** in `NVSHMEMSymmetricMemory.cpp` is an unsynchronized counter used for logging only.

None of these involve Python objects or the Python C API, so they are pure C++ thread-safety issues rather than Python-specific free-threading issues.
