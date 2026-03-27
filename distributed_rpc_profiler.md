# Free-Threading Safety Audit: distributed_rpc_profiler

## Overall Risk: Medium

## Files Audited
- `distributed/rpc/profiler/remote_profiler_manager.cpp`
- `distributed/rpc/profiler/remote_profiler_manager.h`
- `distributed/rpc/profiler/server_process_global_profiler.cpp`
- `distributed/rpc/profiler/server_process_global_profiler.h`

## Detailed Findings

### Issue 1 — RemoteProfilerManager leaky singleton (Low)

**File:** `remote_profiler_manager.cpp`, lines 9-12

```cpp
/*static */ RemoteProfilerManager& RemoteProfilerManager::getInstance() {
  static RemoteProfilerManager* handler = new RemoteProfilerManager();
  return *handler;
}
```

C++ guarantees thread-safe initialization of static locals. The leaky singleton pattern avoids destruction-order issues. However, the constructor accesses `RpcAgent::getCurrentRpcAgent()` which could fail if called before the agent is set. This is a lifecycle issue, not a free-threading issue.

**Risk:** Low.

### Issue 2 — RemoteProfilerManager `currentThreadLocalKey_` (Low)

**File:** `remote_profiler_manager.cpp`, line 8 / `remote_profiler_manager.h`, line 51

```cpp
static thread_local std::optional<std::string> RemoteProfilerManager::currentThreadLocalKey_ = std::nullopt;
```

Thread-local storage. All access methods (`setCurrentKey`, `isCurrentKeySet`, `unsetCurrentKey`, `getCurrentProfilingKey`) operate on this thread-local variable. Safe under free-threading.

**Risk:** Low.

### Issue 3 — RemoteProfilerManager `profiledRpcKeys_` map (Low)

**File:** `remote_profiler_manager.cpp`, lines 33-76

All access to `profiledRpcKeys_` is guarded by `mutex_`:
- `eraseKey()` takes `std::lock_guard<std::mutex>`
- `retrieveRPCProfilingKey()` takes `std::lock_guard<std::mutex>`
- `saveRPCKey()` takes `std::lock_guard<std::mutex>`

The `currentLocalId_` counter is also guarded by the same mutex in `getNextLocalId()`.

**Risk:** Low. Properly synchronized.

### Issue 4 — Global mutable `currentStateStackEntryPtr` (Medium)

**File:** `server_process_global_profiler.h`, lines 71-72 / `server_process_global_profiler.cpp`, lines 15-16

```cpp
mutexType currentStateStackEntryMutex;
std::shared_ptr<StateStackEntry> currentStateStackEntryPtr = nullptr;
```

These are namespace-scope mutable globals. Access is synchronized via a shared/exclusive mutex:
- `StateStackEntry::current()` uses a read lock (`rLockType`)
- `StateStackEntry::pushRange()` uses a write lock (`wLockType`)
- `StateStackEntry::popRange()` uses a write lock (`wLockType`)

The `shared_ptr` reads and writes are protected by the appropriate lock type. This is correctly synchronized.

However, there is a subtlety: the `current()` method returns a `shared_ptr` copy taken under the read lock. The caller then uses this `shared_ptr` after the lock is released. This is safe because the returned `shared_ptr` holds a strong reference, keeping the `StateStackEntry` alive. But if the caller then traverses the linked list via `prevPtr()`, those reads are safe because the `prevPtr_` and `statePtr_` fields are `const` and set at construction time.

**Risk:** Medium. The global mutable state pattern is inherently risky but is properly protected by the shared mutex. The concern is moderate because the `currentStateStackEntryPtr` is a non-Python global that is accessed from RPC processing threads.

### Issue 5 — State class `results_` vector (Low)

**File:** `server_process_global_profiler.h`, lines 34-40 / `server_process_global_profiler.cpp`, lines 7-13

```cpp
void State::pushResult(thread_event_lists result) {
  std::unique_lock<std::mutex> lock(resultsMutex_);
  results_.emplace_back(std::move(result));
}

std::vector<thread_event_lists> State::results() {
  std::unique_lock<std::mutex> lock(resultsMutex_);
  std::vector<thread_event_lists> results;
  results.swap(results_);
  return results;
}
```

Both `pushResult` and `results()` are guarded by `resultsMutex_`. Properly synchronized.

**Risk:** Low.

### Issue 6 — `pushResultRecursive` traversal (Low)

**File:** `server_process_global_profiler.cpp`, lines 37-45

```cpp
void pushResultRecursive(
    std::shared_ptr<StateStackEntry> stateStackEntryPtr,
    const thread_event_lists& result) {
  while (stateStackEntryPtr) {
    stateStackEntryPtr->statePtr()->pushResult(result);
    stateStackEntryPtr = stateStackEntryPtr->prevPtr();
  }
}
```

This traverses the immutable linked list of `StateStackEntry` objects via `prevPtr()`, calling `pushResult()` on each state. Since `prevPtr_` and `statePtr_` are const members set at construction, and `pushResult()` is mutex-protected, this is safe.

**Risk:** Low.

### Issue 7 — macOS fallback to `std::mutex` (Low)

**File:** `server_process_global_profiler.h`, lines 56-68

On macOS, the code falls back from `std::shared_timed_mutex` to `std::mutex` due to older SDK requirements. This reduces concurrency (all locks are exclusive) but does not affect correctness.

**Risk:** Low.

## Summary

The profiler subsystem is generally well-synchronized. The `RemoteProfilerManager` uses thread-local storage for the current profiling key and a mutex for the global profiled keys map. The server process-global profiler uses a shared/exclusive mutex to protect its global state stack. The primary concern (rated Medium) is the global mutable `currentStateStackEntryPtr` pattern, though it is properly locked. No Python C API calls are made in these files, so Python-specific free-threading concerns (critical sections, borrowed refs) do not apply.
