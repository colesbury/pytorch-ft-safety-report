# Free-Threading Safety Audit: lazy_core__part1

## Overall Risk: LOW-MEDIUM

## Files Audited
- `lazy/core/cache.h`
- `lazy/core/config.cpp`
- `lazy/core/config.h`
- `lazy/core/debug_util.cpp`
- `lazy/core/debug_util.h`
- `lazy/core/dynamic_ir.h`
- `lazy/core/hash.cpp`
- `lazy/core/hash.h`
- `lazy/core/helpers.cpp`
- `lazy/core/helpers.h`
- `lazy/core/ir.cpp`
- `lazy/core/ir.h`
- `lazy/core/ir_builder.h`
- `lazy/core/ir_dump_util.cpp`
- `lazy/core/ir_dump_util.h`

## Findings

### Issue 1: Unprotected static std::function for Python frames (debug_util.cpp:57-59)

**Severity:** MEDIUM
**Description:** `GetPythonFramesFunction()` returns a mutable reference to a static `std::function` object. This is written during `initLazyBindings` (in `lazy/python/init.cpp:336`) and read during IR debugging from any thread (`GetMetaDataIfDebugging`, `GetFirstUserFrameInPython`, `DebugUtil::GetTensorsGraphInfo`). Under free-threading, concurrent read/write to `std::function` is a data race.
**Existing Protection:** None. The write happens during module init which is typically serialized, but reads can happen concurrently from any thread performing lazy tensor operations.
**Recommendation:** Use `std::atomic<void*>` with a function pointer, or protect with a mutex, or use `std::call_once` for the assignment.

### Issue 2: getLTCForceFallback() returns reference to unprotected static string (config.cpp:77-88)

**Severity:** LOW
**Description:** `getLTCForceFallback()` returns a mutable `std::string&` to a file-static string. This is read by `force_eager_fallback()` (ts_eager_fallback.cpp) during operation dispatch and written by `_set_force_fallback` from Python. Concurrent read/write to `std::string` is a data race.
**Existing Protection:** The lazy-init lambda runs only once (via `static bool`), but subsequent read/write pairs have no synchronization.
**Recommendation:** Protect with a mutex, or make the setter/getter use atomic operations (e.g., store to a `std::shared_ptr<const std::string>` atomically).

### Issue 3: Static ShapeCache in Node::computeShape (ir.cpp:129)

**Severity:** LOW
**Description:** `static ShapeCache* cache = new ShapeCache(FLAGS_torch_lazy_shape_cache_size);` creates a leaked singleton. The `Cache` class uses internal `std::mutex` for all operations (`Add`, `Get`), so concurrent access is safe.
**Existing Protection:** `Cache` uses `std::mutex` internally.
**Recommendation:** No changes needed. The cache is properly synchronized.

### Issue 4: Static HashCombine constant (hash.cpp:77)

**Severity:** NONE
**Description:** `static const hash_t kb(101, 0x27d4eb2f165667c5);` is a constant. Safe under free-threading.
**Existing Protection:** `const` qualifier.
**Recommendation:** None.

### Issue 5: Static lazy-init patterns in debug_util.cpp

**Severity:** LOW
**Description:** Several patterns use Meyer's singleton for lazy initialization:
- `GetDefaultGraphFormat()` (line 63): `static GraphFormat format = DefaultGraphFormat();` -- thread-safe static init in C++11+.
- `ExperimentEnabled()` (line 167): `static const std::unordered_set<std::string>* xset = LoadExperiments();` -- thread-safe static init.
- `SaveTensorsGraphInfo()` (lines 155-158): `static const std::string save_file = ...` and `static std::mutex lock;` -- both are thread-safe.
**Existing Protection:** C++ guarantees thread-safe initialization of function-local statics.
**Recommendation:** No changes needed.

### Issue 6: C10_DEFINE flags (config.cpp, ir.cpp)

**Severity:** LOW
**Description:** Global flags like `FLAGS_torch_lazy_reuse_ir`, `FLAGS_torch_lazy_ir_debug`, etc. are read from multiple threads and can be modified via Python API (`_set_reuse_ir`, `_set_symbolic_shape_mode`). These are typically `int32_t` or `bool` values implemented by gflags/c10 flag system. Writes to these may race with reads, but since they are simple scalar types, torn reads are unlikely on modern architectures. The risk is stale reads, not crashes.
**Existing Protection:** Scalar reads/writes are typically atomic on modern hardware for aligned types.
**Recommendation:** Acceptable risk for configuration flags. No urgent changes needed.

### Issue 7: Node::enableDynamicShape() static bool (ir.cpp:58-60)

**Severity:** NONE
**Description:** `static bool enabled = c10::utils::has_env("LTC_ENABLE_DYNAMIC_SHAPES");` is initialized once (thread-safe static init) and then only read. The `FLAGS_ltc_enable_dynamic_shapes` check is a separate flag read.
**Existing Protection:** C++ thread-safe static init.
**Recommendation:** None.

## Summary

The main concern is `GetPythonFramesFunction()` which returns a mutable reference to a static `std::function` -- this is a potential data race between the write during module init and concurrent reads during IR tracing. The `getLTCForceFallback()` has a similar but lower-risk pattern. The `Cache` class and other static singletons are properly synchronized with mutexes. Pure computation utilities (hash, helpers, ir_dump_util) are stateless and safe.
