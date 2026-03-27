# Free-Threading Safety Audit: profiler_orchestration

**Overall Risk: MEDIUM**

## Files Reviewed
- `profiler/orchestration/observer.cpp`
- `profiler/orchestration/observer.h`
- `profiler/orchestration/python_tracer.cpp`
- `profiler/orchestration/python_tracer.h`
- `profiler/orchestration/vulkan.cpp`
- `profiler/orchestration/vulkan.h`

## Detailed Findings

### Issue 1 -- Unsynchronized global function pointers for tracer registration (python_tracer.cpp)

**Severity: MEDIUM**
**Location:** `python_tracer.cpp`, lines ~5-6, 33-53

```cpp
namespace {
MakeFn make_fn;
MakeMemoryFn memory_make_fn;
}

void registerTracer(MakeFn make_tracer) {
  make_fn = make_tracer;
}

std::unique_ptr<PythonTracerBase> PythonTracerBase::make(RecordQueue* queue) {
  if (make_fn == nullptr) {
    return std::make_unique<NoOpPythonTracer>();
  }
  return make_fn(queue);
}
```

`make_fn` and `memory_make_fn` are plain function pointers (non-atomic) written by `registerTracer` / `registerMemoryTracer` and read by `PythonTracerBase::make` / `PythonMemoryTracerBase::make`. Under free-threading, if registration happens concurrently with profiler start (which calls `make`), this is a data race.

In practice, registration typically happens during module import (single-threaded), and `make` is called during profiler setup. The risk is moderate because Python module import can be concurrent in free-threaded builds.

**Recommendation:** Use `std::atomic<MakeFn>` and `std::atomic<MakeMemoryFn>` for these function pointers.

### Issue 2 -- Unsynchronized global function pointer for Vulkan (vulkan.cpp)

**Severity: LOW**
**Location:** `vulkan.cpp`, lines ~7-16

```cpp
namespace {
GetShaderNameAndDurationNsFn get_shader_name_and_duration_ns_fn;
}

void registerGetShaderNameAndDurationNs(
    GetShaderNameAndDurationNsFn get_shader_name_and_duration_ns) {
  get_shader_name_and_duration_ns_fn =
      std::move(get_shader_name_and_duration_ns);
}
```

`get_shader_name_and_duration_ns_fn` is a `std::function` that is written during registration and read during profiling. Unlike the tracer function pointers (which are plain pointers), this is a `std::function` object, making the data race more dangerous since `std::function` assignment is not atomic.

The code comment says deregistration only happens in the QueryPool destructor after profiling finishes, so in practice the write/read window does not overlap. Still, under free-threading this assumption is fragile.

**Recommendation:** Protect with a mutex or use an atomic pointer to a `std::function`.

### Issue 3 -- `GlobalStateManager` used for profiler state (observer.cpp)

**Severity: MEDIUM**
**Location:** `observer.cpp`, lines ~9, 126-164

```cpp
using GlobalManager = GlobalStateManager<ProfilerStateBase>;
```

As noted in the profiler__part1 report, `GlobalStateManager` has no internal synchronization on its `state_` member. `ProfilerStateBase::get(/*global=*/true)` calls `GlobalManager::get()` which reads `state_`, while `ProfilerStateBase::push()` writes it. Under free-threading, concurrent profiler enable/disable from different threads racing with `get()` calls is a data race.

`ProfilerStateBase` does have a `state_mutex_` member, but this protects the profiler state internals, not the `GlobalStateManager`'s pointer to the state itself.

### Issue 4 -- `ProfilerStateBase::setCallbackHandle` / `removeCallback` (observer.cpp)

**Severity: LOW**
**Location:** `observer.cpp`, lines ~166-183

```cpp
void ProfilerStateBase::setCallbackHandle(at::CallbackHandle handle) {
  if (handle_) {
    at::removeCallback(handle_);
    ...
  }
  handle_ = handle;
}
```

`handle_` is a plain `at::CallbackHandle` (uint64_t). Under free-threading, if `setCallbackHandle` and `removeCallback` are called concurrently, this is a data race. However, these are typically called in a single-threaded profiler setup/teardown context.

### Issue 5 -- `profilerEnabled()` / `profilerType()` / `getProfilerConfig()` (observer.cpp)

**Severity: LOW**
**Location:** `observer.cpp`, lines ~185-202

These functions call `ProfilerStateBase::get(/*global=*/false)` which reads from thread-local debug info (safe per-thread), and `get(/*global=*/true)` which reads from the unsynchronized `GlobalStateManager`. The global path has the same issue as Issue 3 above.

## Summary

The primary concerns are the unprotected global function pointers for tracer registration (Issue 1) and the `GlobalStateManager` synchronization gap (Issue 3). The Vulkan registration (Issue 2) is lower risk because of the narrow write/read window. Most profiler state access goes through thread-local storage (which is safe per-thread), but the global profiler state path through `GlobalStateManager` needs synchronization for free-threading.
