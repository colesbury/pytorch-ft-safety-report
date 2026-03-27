# Free-Threading Safety Audit: profiler__part1

**Overall Risk: MEDIUM**

## Files Reviewed
- `profiler/api.h`
- `profiler/collection.cpp`
- `profiler/collection.h`
- `profiler/combined_traceback.cpp`
- `profiler/combined_traceback.h`
- `profiler/containers.h`
- `profiler/data_flow.cpp`
- `profiler/data_flow.h`
- `profiler/events.h`
- `profiler/kineto_client_interface.cpp`
- `profiler/kineto_client_interface.h`
- `profiler/kineto_shim.cpp`
- `profiler/kineto_shim.h`
- `profiler/perf.cpp`
- `profiler/perf.h`
- `profiler/perf-inl.h`
- `profiler/util.cpp`
- `profiler/util.h`

## Detailed Findings

### Issue 1 -- Global mutable `std::function` state without synchronization (collection.cpp)

**Severity: MEDIUM**
**Location:** `collection.cpp`, lines ~1620-1690

Four feature-flag globals follow the same pattern -- a function-local static `std::function<bool()>` that is read and written without any locking:

```cpp
std::function<bool()>& record_concrete_inputs_enabled_fn() {
  static std::function<bool()> fn = []() { return true; };
  return fn;
}
bool get_record_concrete_inputs_enabled() {
  return record_concrete_inputs_enabled_fn()();
}
void set_record_concrete_inputs_enabled_fn(std::function<bool()> fn) {
  record_concrete_inputs_enabled_fn() = std::move(fn);
}
```

The same pattern exists for `fwd_bwd_enabled_fn`, `cuda_sync_enabled_fn`, and `record_tensor_addrs_enabled`. Any concurrent call to a getter while a setter is in progress is a data race on a non-trivial `std::function` object. Additionally, `get_record_tensor_addrs_enabled` caches into a function-local `static std::optional<bool>` which is itself a race.

**Recommendation:** Use `std::atomic<bool>` for the simple value paths, or protect with a mutex. The `std::function<bool()>` overloads could use a `std::shared_mutex` for read-heavy access.

### Issue 2 -- Static atomic counter for correlation IDs (collection.cpp)

**Severity: LOW**
**Location:** `collection.cpp`, line ~299

```cpp
static std::atomic<uint64_t> counter_{0};
```

Inside `EventBlock` constructor. This is already atomic, so it is thread-safe. No action needed.

### Issue 3 -- Global `queue_id_` and thread_local `sub_queue_cache_` (collection.cpp)

**Severity: LOW**
**Location:** `collection.cpp`, lines ~560-561

```cpp
std::atomic<uint32_t> queue_id_{0};
thread_local SubQueueThreadCache sub_queue_cache_{0, nullptr};
```

`queue_id_` is atomic (safe). `sub_queue_cache_` is `thread_local` (safe per-thread). The `getSubqueue()` method then takes `sub_queue_mutex_` before touching the shared `sub_queues_` map. This is correctly synchronized.

### Issue 4 -- `soft_assert_raises_` global mutable state (util.cpp)

**Severity: LOW**
**Location:** `util.cpp`, lines ~19-28

```cpp
namespace {
std::optional<bool> soft_assert_raises_;
}
void setSoftAssertRaises(std::optional<bool> value) {
  soft_assert_raises_ = value;
}
bool softAssertRaises() {
  return soft_assert_raises_.value_or(false);
}
```

Unprotected `std::optional<bool>`. Concurrent set/get is a data race on the optional's engaged flag. In practice this is only set during configuration and read during profiling, so the risk is low.

**Recommendation:** Use `std::atomic<bool>` plus a separate `std::atomic<bool>` for the "has value" flag, or simply an `std::atomic<int>` with a sentinel.

### Issue 5 -- `GlobalStateManager` singleton pattern (util.h)

**Severity: MEDIUM**
**Location:** `util.h`, lines ~140-169

```cpp
template <typename T>
class GlobalStateManager {
  static GlobalStateManager& singleton() {
    static GlobalStateManager singleton_;
    return singleton_;
  }
  static void push(std::shared_ptr<T>&& state) { ... }
  static auto* get() { return singleton().state_.get(); }
  static std::shared_ptr<T> pop() { ... }
private:
  std::shared_ptr<T> state_;
};
```

The `state_` shared_ptr is read/written from `push`, `get`, `pop` with no synchronization. If two threads concurrently call `push` and `get` on the same `GlobalStateManager<ProfilerStateBase>` or `GlobalStateManager<ExecutionTraceObserver>`, this is a data race. This affects `ProfilerStateBase::get(/*global=*/true)` and `ProfilerStateBase::push()`.

**Recommendation:** Add a mutex to `GlobalStateManager` or use `std::atomic<std::shared_ptr<T>>` (C++20).

### Issue 6 -- `RegisterLibKinetoClient` static initialization (kineto_client_interface.cpp)

**Severity: LOW**
**Location:** `kineto_client_interface.cpp`, lines ~105-113

```cpp
struct RegisterLibKinetoClient {
  RegisterLibKinetoClient() {
    if constexpr (kEnableGlobalObserver) {
      static profiler::impl::LibKinetoClient client;
      libkineto::api().registerClient(&client);
    }
  }
} register_libkineto_client;
```

This runs during static initialization (single-threaded), so the static local is safely initialized. The `registerClient` call may have its own thread-safety implications in libkineto, but that is outside this audit scope.

### Issue 7 -- `CapturedTraceback::addPythonUnwinder` lock-free list (combined_traceback.cpp)

**Severity: LOW**
**Location:** `combined_traceback.cpp`, lines ~190-195

```cpp
static std::atomic<CapturedTraceback::Python*> python_support_ = nullptr;

void CapturedTraceback::addPythonUnwinder(CapturedTraceback::Python* p) {
  CapturedTraceback::Python* old_unwinder = python_support_.load();
  do {
    p->next_ = old_unwinder;
  } while (!python_support_.compare_exchange_strong(old_unwinder, p));
}
```

This is a correct lock-free singly-linked list insertion using CAS. The `gather()` method loads the head atomically. The `next_` pointer is set before publication via CAS. This is safe.

### Issue 8 -- `EventTable` static const map (perf.cpp)

**Severity: LOW**
**Location:** `perf.cpp`, lines ~30-49

This is a `static const` map initialized at static-init time. Read-only after initialization. Safe.

### Issue 9 -- Static const `ActivityTypeMap` tables (kineto_shim.cpp)

**Severity: LOW**
**Location:** `kineto_shim.cpp`, lines ~22-80

All `const` maps at namespace scope. Read-only. Safe.

## Summary

The primary concern in this group is the unprotected global `std::function<bool()>` feature flags in `collection.cpp` (Issue 1) and the unsynchronized `GlobalStateManager::state_` shared_ptr (Issue 5). Both are called from profiler setup/query paths that can run on multiple threads under free-threading. The remaining issues are low severity -- either already using atomics, or are configuration-time-only writes.
