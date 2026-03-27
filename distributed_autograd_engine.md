# Free-Threading Safety Audit: distributed_autograd_engine

## Overall Risk: Low

## Files Audited
- `distributed/autograd/engine/dist_engine.cpp`
- `distributed/autograd/engine/dist_engine.h`

## Findings

### 1. Leaky singleton `getInstance` -- No Issue
**File:** `distributed/autograd/engine/dist_engine.cpp`
**Lines:** 146-149

```cpp
static DistEngine* engine = new DistEngine();
return *engine;
```

Function-local static with C++11 thread-safe initialization guarantee. Heap-allocated and never deleted (leaky singleton). Safe.

### 2. `initializedContextIds_` protected by `initializedContextIdsLock_` -- No Issue
**File:** `distributed/autograd/engine/dist_engine.h`
**Lines:** 141-143

```cpp
std::unordered_set<int64_t> initializedContextIds_;
mutable std::mutex initializedContextIdsLock_;
```

All accesses to `initializedContextIds_` are guarded by `initializedContextIdsLock_`:

- `executeSendFunctionAsync`: locks at line 481, unlocks at 492 or 550
- `execute`: locks at line 583
- `cleanupBackwardPass`: locks at line 628
- `numBackwardPasses`: locks at line 633

Safe.

### 3. `global_cpu_ready_queue_` shared across threads -- No Issue
**File:** `distributed/autograd/engine/dist_engine.cpp`
**Lines:** 117-118

The `global_cpu_ready_queue_` is a `shared_ptr<ReadyQueue>` created in the constructor and shared with the `global_cpu_thread_`. `ReadyQueue` is internally thread-safe (it uses its own condition variable and mutex for push/pop). The queue is only written to via `push` and read via `pop`, both of which are internally synchronized. Safe.

### 4. `engine_` reference to the autograd engine -- No Issue
**File:** `distributed/autograd/engine/dist_engine.h`
**Line:** 146

```cpp
torch::autograd::Engine& engine_;
```

This is a reference to the global autograd `Engine` singleton obtained via `Engine::get_default_engine()`. The `Engine` has its own internal synchronization. Safe.

### 5. `TORCH_ASSERT_NO_GIL_WITHOUT_PYTHON_DEP` in destructor -- No Issue
**File:** `distributed/autograd/engine/dist_engine.cpp`
**Line:** 141

This is a debug assertion that checks the GIL is not held during destruction, to avoid deadlocks when joining the CPU thread. Under free-threading, this assertion may need updating (the GIL concept doesn't exist), but it is a debug-only check and not a safety issue.

### 6. Static string constants -- No Issue
**File:** `distributed/autograd/engine/dist_engine.cpp`
**Lines:** 28-29

```cpp
static constexpr const char* kNumBackwardPasses = "num_current_backward_passes";
static constexpr const char* kNumAutogradContexts = "num_autograd_contexts";
```

Compile-time constants. Safe.

### 7. `DistAccumulateGradCaptureHook` captures `autogradContext_` -- No Issue
**File:** `distributed/autograd/engine/dist_engine.cpp`
**Lines:** 35-74

The hook stores a `ContextPtr` (shared_ptr) to the autograd context. When invoked, it calls `autogradContext_->accumulateGrad()` which acquires the context's internal mutex. The shared_ptr keeps the context alive. Safe.

### 8. `computeDependencies` accesses context methods -- No Issue
**File:** `distributed/autograd/engine/dist_engine.cpp`
**Lines:** 182-346

This method calls `autogradContext->sendFunctions()` which returns a copy of the map under the context's lock, and `autogradContext->setGraphTask()` which acquires the context's lock. All context access is properly synchronized. Safe.

## Summary

The distributed autograd engine is well-synchronized. It uses a leaky singleton pattern, a dedicated mutex for the initialized context IDs set, and delegates to the `DistAutogradContext` (which has its own mutex) and the vanilla autograd `Engine` (which has its own thread pool and synchronization). The `ReadyQueue` used for CPU continuations is internally thread-safe. No free-threading issues were identified.
