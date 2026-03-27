# Free-Threading Safety Audit: profiler_standalone

**Overall Risk: LOW**

## Files Reviewed
- `profiler/standalone/execution_trace_observer.cpp`
- `profiler/standalone/execution_trace_observer.h`
- `profiler/standalone/itt_observer.cpp`
- `profiler/standalone/itt_observer.h`
- `profiler/standalone/nvtx_observer.cpp`
- `profiler/standalone/nvtx_observer.h`
- `profiler/standalone/privateuse1_observer.cpp`
- `profiler/standalone/privateuse1_observer.h`
- `profiler/standalone/privateuse1_profiler.cpp`
- `profiler/standalone/privateuse1_profiler.h`

## Detailed Findings

### Issue 1 -- `ExecutionTraceObserver` shared state with `gMutex` (execution_trace_observer.cpp)

**Severity: LOW**
**Location:** `execution_trace_observer.cpp`, lines ~108-179

```cpp
struct ExecutionTraceObserver {
  std::map<size_t, std::stack<ID>> opStack;
  std::map<const void*, ID> objectId;
  std::recursive_mutex gMutex;
  ...
  std::atomic<ID> id_{2};
};
```

The `ExecutionTraceObserver` uses a `std::recursive_mutex gMutex` to protect its shared state (`opStack`, `objectId`, `out`). This is correct for multi-threaded access. The `id_` counter is atomic. The overall design is thread-safe.

The `ExecutionTraceObserver` is managed through `GlobalStateManager<ExecutionTraceObserver>`, which has the same unsynchronized shared_ptr issue noted in the profiler__part1 report. However, the observer is typically created/destroyed from a single thread (the Python main thread).

### Issue 2 -- `ITTThreadLocalState` and `NVTXThreadLocalState` are per-thread (itt_observer.cpp, nvtx_observer.cpp)

**Severity: LOW**
**Location:** `itt_observer.cpp`, `nvtx_observer.cpp`

Both `ITTThreadLocalState` and `NVTXThreadLocalState` extend `ProfilerStateBase` and are pushed into thread-local debug info via `c10::ThreadLocalDebugInfo::_push`. The `getTLS()` static method retrieves the per-thread state. This is inherently thread-safe because each thread has its own copy.

The `NVTXThreadLocalState::producer_tensor_map_` is a per-instance `std::unordered_map` that is only accessed from the thread that owns the state (via enter/exit callbacks on that thread). Safe.

### Issue 3 -- `pushPRIVATEUSE1CallbacksStub` global (privateuse1_observer.cpp)

**Severity: LOW**
**Location:** `privateuse1_observer.cpp`, line ~5; `privateuse1_observer.h`, lines ~10-34

```cpp
PushPRIVATEUSE1CallbacksStub pushPRIVATEUSE1CallbacksStub;
```

This global struct holds a `CallBackFnPtr push_privateuse1_callbacks_fn` that is set via `set_privateuse1_dispatch_ptr` (during static initialization via `REGISTER_PRIVATEUSE1_OBSERVER`) and read via `operator()`. The write happens at static init time and reads happen later. Under free-threading, if registration is deferred past static init (e.g., during module import), there could be a race. In practice this is extremely unlikely.

### Issue 4 -- `PrivateUse1ProfilerRegistry` singleton with mutex (privateuse1_profiler.cpp)

**Severity: LOW**
**Location:** `privateuse1_profiler.cpp`, lines ~18-69

```cpp
PrivateUse1ProfilerRegistry& PrivateUse1ProfilerRegistry::instance() {
  static PrivateUse1ProfilerRegistry registry;
  return registry;
}
```

The singleton is safely initialized (C++11 guarantees). All methods (`registerFactory`, `hasFactory`, `isRegisteredWithKineto`, `onKinetoInit`) acquire `mutex_` before accessing shared state. This is correctly synchronized for free-threading.

### Issue 5 -- Static `CUDAMethods` and `ITTMethods` registration (stubs referenced from standalone)

**Severity: LOW**

The NVTX and ITT observers call into `cudaStubs()` and `ittStubs()` which return pointers set during static initialization. These are set once and then read-only. Safe.

## Summary

The standalone profiler components are generally well-protected. The `ExecutionTraceObserver` uses a recursive mutex for its shared state, the `PrivateUse1ProfilerRegistry` uses a proper mutex, and the NVTX/ITT observers use thread-local state. The only inherited concern is the `GlobalStateManager` pattern (from `profiler__part1`), which affects observer lifecycle management but is low-risk in practice since observers are typically managed from a single thread.
