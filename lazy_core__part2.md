# Free-Threading Safety Audit: lazy_core__part2

## Overall Risk: LOW

## Files Audited
- `lazy/core/ir_metadata.cpp`
- `lazy/core/ir_metadata.h`
- `lazy/core/ir_util.cpp`
- `lazy/core/ir_util.h`
- `lazy/core/lazy_graph_executor.cpp`
- `lazy/core/lazy_graph_executor.h`
- `lazy/core/metrics.cpp`
- `lazy/core/metrics.h`
- `lazy/core/multi_wait.cpp`
- `lazy/core/multi_wait.h`
- `lazy/core/permutation_util.cpp`
- `lazy/core/permutation_util.h`
- `lazy/core/shape.cpp`
- `lazy/core/shape.h`
- `lazy/core/shape_inference.cpp`

## Findings

### Issue 1: thread_local ScopeContext (ir_metadata.cpp:47)

**Severity:** NONE
**Description:** `thread_local ScopeContext g_scope_context;` is per-thread state. Each thread has its own scope context. This is inherently safe under free-threading.
**Existing Protection:** `thread_local` storage.
**Recommendation:** None.

### Issue 2: LazyGraphExecutor global registry (lazy_graph_executor.cpp:61)

**Severity:** LOW
**Description:** `std::atomic<LazyGraphExecutor*> lazy_graph_executor_registry;` is accessed via `Register()` and `Get()`. The atomic ensures visibility, but the pointed-to object must be fully constructed before being stored. `Register()` is called during `InitTorchScriptBackend()`, which is typically called once during setup.
**Existing Protection:** `std::atomic` store/load.
**Recommendation:** Safe as-is. The store uses relaxed ordering implicitly but the object lifetime is managed by static allocation.

### Issue 3: DeviceContextArena singleton and per-device mutexes (lazy_graph_executor.cpp:64-190)

**Severity:** LOW
**Description:** `DeviceContextArena::Get()` uses a leaked Meyer's singleton. The arena maintains per-device contexts protected by individual mutexes (`devctx->lock`). The arena's own `lock_` protects the `device_contexts_` map. This is a well-structured concurrent design.
**Existing Protection:** `std::mutex` at both arena level and per-device level.
**Recommendation:** No changes needed. This is well-synchronized.

### Issue 4: DeviceLockerArena and DeviceLocker (lazy_graph_executor.cpp:201-276)

**Severity:** NONE
**Description:** `DeviceLocker` uses `std::mutex` and `std::condition_variable` for lock/unlock/barrier operations. `DeviceLockerArena` protects its map with a mutex.
**Existing Protection:** Full mutex-based synchronization.
**Recommendation:** None.

### Issue 5: DataCacheArena and device caches (lazy_graph_executor.cpp:278-346)

**Severity:** LOW
**Description:** `DataCacheArena::GetDataCache()` uses a mutex to protect `device_caches_`. The returned `DataCache*` pointer is stable (stored in `std::unique_ptr` in the map). The `Cache` class itself is mutex-protected. However, `GetDeviceData()` (line 287-300) calls `Get()->GetDataCache(device)` which acquires the arena mutex, then operates on the cache (which has its own mutex). The static `s_empty_cache` (line 343) is created once and is zero-sized so all operations on it are no-ops.
**Existing Protection:** Two-level mutex synchronization.
**Recommendation:** No changes needed.

### Issue 6: MetricsArena (metrics.cpp/h)

**Severity:** NONE
**Description:** `MetricsArena` uses `std::mutex lock_` to protect metrics/counters maps. `MetricData` uses its own `std::mutex` for sample operations. `CounterData` uses `std::atomic<int64_t>`. The lazy init patterns in `Metric::GetData()` and `Counter::GetData()` use `std::atomic` loads with `RegisterMetric`/`RegisterCounter` as synchronization points. Multiple threads may enter the init path simultaneously, but `RegisterMetric` uses a lock and checks `if (*data == nullptr)`.
**Existing Protection:** Properly synchronized with atomics and mutexes.
**Recommendation:** None.

### Issue 7: Metric/Counter lazy init race (metrics.cpp:289-302, 306-318)

**Severity:** LOW
**Description:** In `Metric::GetData()`, there is a TOCTOU pattern: the atomic load of `data_` (line 290), then `RegisterMetric` (line 295), then store to `data_` (line 299). If two threads race here, both may enter the `if` block, but `RegisterMetric` is synchronized and will return the same `data_ptr_`, so both threads store the same pointer. The `data_ptr_` (a `std::shared_ptr`) is being written by the arena under its lock, but read outside the lock on line 298. Since `shared_ptr` assignment is not atomic, this is technically a race on `data_ptr_`.
**Existing Protection:** The atomic `data_` field and the arena's mutex partially protect this, but `data_ptr_` (a mutable `shared_ptr`) can race.
**Recommendation:** The `data_ptr_` member should be protected -- either always access under a lock, or restructure to avoid the double-checked locking pattern on a non-atomic shared_ptr.

### Issue 8: MultiWait (multi_wait.cpp/h)

**Severity:** NONE
**Description:** Fully mutex/condition-variable protected. Safe.
**Existing Protection:** `std::mutex` and `std::condition_variable`.
**Recommendation:** None.

### Issue 9: Static init patterns in shape.cpp

**Severity:** NONE
**Description:** `symbolicShapeEnabled()` (shape.cpp:59-62) uses `static bool enabled = ...` which is thread-safe C++ static init. The flag check is a scalar read.
**Existing Protection:** Thread-safe static init.
**Recommendation:** None.

### Issue 10: Stateless utility functions

**Severity:** NONE
**Description:** `ir_util.cpp`, `permutation_util.cpp`, and `shape_inference.cpp` are pure computation functions with no static/global mutable state.
**Existing Protection:** Stateless.
**Recommendation:** None.

## Summary

The lazy_graph_executor is well-designed for concurrent access with per-device mutexes, arena-level locks, and atomic registries. The metrics system uses atomics and mutexes appropriately, with a minor concern about the `data_ptr_` shared_ptr in `Metric::GetData()` / `Counter::GetData()`. Most other files are stateless utilities or use thread-local storage. No Python C API is used in these files.
