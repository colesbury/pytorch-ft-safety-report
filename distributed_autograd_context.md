# Free-Threading Safety Audit: distributed_autograd_context

## Overall Risk: Low

## Files Audited
- `distributed/autograd/context/container.cpp`
- `distributed/autograd/context/container.h`
- `distributed/autograd/context/context.cpp`
- `distributed/autograd/context/context.h`

## Findings

### 1. Thread-local `current_context_id_` -- No Issue
**File:** `distributed/autograd/context/container.cpp`
**Line:** 17

```cpp
static thread_local int64_t current_context_id_ = kInvalidContextId;
```

This is `thread_local` and only accessed by the owning thread. All reads and writes (`hasValidContext`, `currentContext`, `newContext`, `eraseContextIdAndReset`, `setCurrentContextId`, `forceCurrentContextId`, `clearCurrentContext`, `currentContextId`) operate on the calling thread's own copy. Safe.

### 2. Static `dist_container_init_lock_` mutex -- No Issue
**File:** `distributed/autograd/context/container.cpp`
**Line:** 19

```cpp
static std::mutex dist_container_init_lock_;
```

This static mutex guards `DistAutogradContainer::init()`. It is a `std::mutex`, which is safe under free-threading. The initialization of the static mutex itself is safe (zero-initialized POD-like on all major implementations).

### 3. Leaky singleton `getInstanceInternal` -- No Issue
**File:** `distributed/autograd/context/container.cpp`
**Lines:** 94-99

```cpp
static DistAutogradContainer* container =
    new DistAutogradContainer(computeNumShards());
```

This is a function-local static variable. C++11 guarantees thread-safe initialization of function-local statics via a compiler-generated guard variable. The object is heap-allocated and never deleted (leaky singleton), so there is no destruction race. Safe.

### 4. `next_context_id_` and `next_autograd_message_id_` atomics -- Low Risk
**File:** `distributed/autograd/context/container.h`
**Lines:** 141, 156

```cpp
std::atomic<int64_t> next_context_id_;
std::atomic<int64_t> next_autograd_message_id_;
```

These are `std::atomic` and used with `++` (sequential consistency by default). Safe.

However, `newAutogradMessageId()` (container.cpp line 101-105) does a non-atomic read of `max_id_` in its `TORCH_INTERNAL_ASSERT` check. Since `max_id_` is only written during `init()` (which is guarded by `dist_container_init_lock_` and happens-before any calls to `newAutogradMessageId()`), this is safe in practice.

### 5. Sharded context map with per-shard locks -- No Issue
**File:** `distributed/autograd/context/container.h`
**Lines:** 112-118, and container.cpp (various methods)

The `autograd_contexts_` vector of `ContextsShard` structs, each with its own `std::mutex lock` and `std::unordered_map<int64_t, ContextPtr> contexts`, is properly locked in all access paths:

- `getOrCreateContext`: locks shard (line 109)
- `newContext`: locks shard (line 142)
- `currentContext`: locks shard (line 167)
- `releaseContextIfPresent`: locks shard (line 178)
- `releaseContext`: locks shard (line 197)
- `isValidContext`: locks shard (line 276)
- `retrieveContext`: locks shard (line 285)
- `numAutogradContexts`: locks each shard (line 315)

All shard access is properly synchronized. Safe.

### 6. `DistAutogradContext` internal mutex -- No Issue
**File:** `distributed/autograd/context/context.cpp` and `context.h`

All mutable operations on `DistAutogradContext` are protected by `lock_` (a `mutable std::mutex`):

- `getKnownWorkerIds`: guarded (line 22)
- `addKnownWorkerId`: guarded (line 27)
- `addSendFunction`: guarded (line 36)
- `addRecvFunction`: guarded (line 48)
- `sendFunctions`: guarded (line 57), returns copy
- `recvFunctions`: guarded (line 62), returns copy
- `accumulateGrad`: guarded (line 74)
- `retrieveGraphTask`: guarded (line 112)
- `setGraphTask`: guarded (line 119)
- `resetGraphTask`: guarded (line 127)
- `addOutstandingRpc`: guarded (line 149)
- `clearOutstandingRpcs`: guarded (line 154)
- `clearAndWaitForOutstandingRpcsAsync`: guarded (line 176)
- `retrieveSendFunction`: guarded (line 223)
- `getGradients`: guarded (line 234)
- `runGradCallbackForVariable`: guarded (lines 249, 257)

The `contextId_` field is `const` and safe to read concurrently.

### 7. `recordGradEvent` called under lock -- No Issue
**File:** `distributed/autograd/context/context.cpp`
**Lines:** 158-172

`recordGradEvent` accesses `gradReadyEvents_` which is always called under `lock_` (from `accumulateGrad` on line 107 and `runGradCallbackForVariable` on line 261). Safe.

### 8. Thread-local `tl_context_ptr` -- No Issue
**File:** `distributed/autograd/context/context.cpp`
**Line:** 266

```cpp
thread_local ContextPtr tl_context_ptr;
```

This is `thread_local` and only accessed by the owning thread via `ThreadLocalDistAutogradContext`. Safe.

### 9. `addOutstandingRpc` callback captures `this` -- Low Risk
**File:** `distributed/autograd/context/context.cpp`
**Lines:** 131-151

The callback in `addOutstandingRpc` captures `this` (the `DistAutogradContext`). The callback accesses `lock_` and `graphTask_`. This is safe as long as the `DistAutogradContext` outlives the future callbacks. Since contexts are reference-counted (`shared_ptr`) and the future holds a reference through the `addOutstandingRpc` list, the context should remain alive. The mutex access within the callback is correct.

## Summary

This group implements the core distributed autograd context management. The code is well-designed for concurrency: it uses a sharded map with per-shard locks for the container, per-context mutexes for individual context state, thread-local storage for the current context ID, atomics for ID generation, and a properly guarded singleton. No free-threading issues were identified. The existing synchronization is independent of the GIL.
