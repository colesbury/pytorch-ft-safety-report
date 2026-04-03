# `pyProfileFn` teardown race: worker callbacks race with `disableProfiler`

- **Status:** Fix pending — [pytorch/pytorch#179262](https://github.com/pytorch/pytorch/pull/179262)
- **Severity:** SEVERE
- **Component:** autograd/profiler_python.cpp, autograd/profiler_kineto.cpp, profiler/collection.cpp
- **Confirmed by TSAN:** Yes (20 of 20 races in `tsan.log.105601`, 111 in reproduction)

## Shared state

Per-thread `ThreadLocalSubqueue` objects owned by `RecordQueue::sub_queues_`,
and per-thread `ThreadLocalResults` objects owned by
`PythonTracer::thread_local_results_` (a `std::deque`). Both are read/destroyed
by the main thread during teardown while worker threads may still be writing to
them from inside `pyProfileFn`.

## Writers

Worker threads call `PythonTracer::pyProfileFn()` (profiler_python.cpp), which:
1. Constructs a `CodeLocation` from the frame, calling `PyFrame_GetLineNumber`
   and `PyFrame_GetCode`.
2. Calls `TraceKeyCacheState::intern()` which on cache miss calls
   `ValueCache::store<>()` (may call into Python via `py::repr()`,
   `.attr("_parameters")`, etc.).
3. Calls `RecordQueue::getSubqueue()` (collection.cpp:722) to get or create a
   per-thread `ThreadLocalSubqueue`, then appends profiling events to it.
4. Accesses `ThreadLocalResults` fields (the `TraceKeyCacheState` tuple,
   `ValueCache`, exit times) to record call/return data.

## Readers / destroyers

The main thread calls `disableProfiler()` (profiler_kineto.cpp:874), which:
1. `state_ptr->removeCallback()` — removes torch op recording callback.
2. `finalizeTrace()` (profiler_kineto.cpp:454):
   a. `recordQueue.stop()` → `PythonTracer::stop()` — calls
      `setprofileAllThreads(nullptr)` which uses `PyEval_SetProfileAllThreads`
      (includes stop-the-world on free-threaded builds).
   b. `recordQueue.getRecords()` (collection.cpp) — iterates `sub_queues_`
      and reads each `ThreadLocalSubqueue`'s events.
   c. `python_tracer_->getEvents()` — post-processes `ThreadLocalResults`.
3. Eventually `PythonTracer` is destroyed, which destroys the
   `thread_local_results_` deque and all `ThreadLocalResults` objects.

## Root cause: `PyCode_Addr2Line` critical section causes thread detach

`PyEval_SetProfileAllThreads(nullptr)` uses stop-the-world (STW) to clear the
profiling callback on all threads atomically. STW waits for all threads to
reach a safe point before proceeding. Threads that are in pure C code (no
Python API calls) stay ATTACHED and won't be parked until they return to the
eval loop.

However, `pyProfileFn` calls CPython C API functions that **take critical
sections on shared Python objects**. The specific call chain is:

```
CodeLocation(PyFrameObject* frame)
  → PyFrame_GetLineNumber(frame)
    → PyUnstable_InterpreterFrame_GetLine(frame->f_frame)
      → PyCode_Addr2Line(code, ...)
        → Py_BEGIN_CRITICAL_SECTION(code)    ← locks the CODE OBJECT
          → _PyCriticalSection_BeginSlow
            → PyMutex_Lock (with _PY_LOCK_DETACH)
              → _PyParkingLot_Park           ← thread DETACHES here
```

**Code objects are shared across all threads executing the same function.**
When multiple worker threads profile calls within the same code object (e.g.,
all threads calling the same `work()` function), they contend on the code
object's mutex inside `PyCode_Addr2Line`. The `PyMutex_Lock` with
`_PY_LOCK_DETACH` flag causes the waiting thread to **detach** from the
interpreter while waiting for the lock.

Once a thread is in the DETACHED state, `park_detached_threads()` in the STW
machinery can CAS it from DETACHED to SUSPENDED immediately — without waiting
for the thread to finish its `pyProfileFn` callback. When STW ends, the thread
re-attaches, acquires the code object lock, and **continues executing
`pyProfileFn`** — writing to `ThreadLocalResults` and subqueues. But by this
time, the main thread has already proceeded to `getRecords()` and is reading
the same data.

## Race scenario (detailed)

1. Worker thread enters `pyProfileFn` for a `PyTrace_CALL` event.
2. `recordPyCall` constructs a `CodeLocation` from `f_back` (the caller frame).
3. `CodeLocation` constructor calls `PyFrame_GetLineNumber(f_back)`.
4. `PyFrame_GetLineNumber` → `PyCode_Addr2Line` → `Py_BEGIN_CRITICAL_SECTION`
   on the code object. The lock is contended (another thread holds it).
5. `PyMutex_Lock` with `_PY_LOCK_DETACH` causes the thread to **detach**.
6. Main thread calls `setprofileAllThreads(nullptr)` → STW begins.
7. STW's `park_detached_threads()` sees the worker in DETACHED state and CAS's
   it to SUSPENDED.
8. STW clears `c_profilefunc = NULL` on all threads, bumps monitoring version.
9. STW calls `_PyEval_StartTheWorld` → all threads resume.
10. Worker thread wakes up from parking lot wait, acquires the code object lock,
    finishes `PyCode_Addr2Line`, continues `pyProfileFn` — writing profiling
    data to `ThreadLocalResults` and subqueues.
11. Main thread proceeds to `getRecords()` → reads the same subqueue data.
12. **Data race** between worker's writes (step 10) and main thread's reads
    (step 11).

## TSAN evidence

Reproduced with 111 TSAN data race reports in a single test run.

Confirmed with CPython instrumentation: a `stw_debug_flag` field added to
`_PyThreadStateImpl` is set on entry to `pyProfileFn` and cleared on exit.
Logging in `detach_thread()` when this flag is set produced thousands of
backtraces per test run, all showing the same call chain through
`PyFrame_GetLineNumber` → `PyCode_Addr2Line` → `_PyCriticalSection_BeginSlow`
→ `PyMutex_Lock` → `_PyParkingLot_Park`.

## Relationship to other issues

- **[GC callback UAF](gc-callback-use-after-free-of-pythontracer.md):**
  Same root cause (teardown without waiting for callbacks) but via a different
  callback path (GC callback vs `pyProfileFn`). The GC callback race is harder
  to trigger; the `pyProfileFn` race is the common case confirmed by TSAN.
- **[Shared ValueCache](shared-valuecache-across-threads.md) (FIXED):** The
  #178552 fix made `ValueCache` per-thread, eliminating concurrent writes
  *between* worker threads. But it does not fix this race: the main thread
  still destroys the per-thread `ValueCache` (inside `ThreadLocalResults`)
  while the owning worker thread is still using it.
- **[thread_local_results_map](profiler-thread-local-results-map-race.md):**
  The map lookups are safe (read-only after init), but the `ThreadLocalResults`
  objects the map points to are destroyed by this race.

## Suggested fix

The core problem is that `pyProfileFn` calls CPython C API functions that can
detach the thread, making STW insufficient to ensure callbacks have completed.

**Option 1: Avoid `PyFrame_GetLineNumber` in the profiling callback.**
`CodeLocation` calls `PyFrame_GetLineNumber` which triggers
`PyCode_Addr2Line` with a critical section on the shared code object. Instead,
defer line number resolution to post-processing (in `getEvents()`), where
there is no concurrency. During profiling, store only the frame/code pointer
and the bytecode offset (`_PyInterpreterFrame_LASTI`), which can be read
without locking.

**Option 2: Barrier after STW.** After `setprofileAllThreads(nullptr)`, spin
until all threads have exited `pyProfileFn`. This requires a per-thread flag
(not a shared atomic counter, which would add cross-thread synchronization).
The `stw_debug_flag` mechanism demonstrated in this investigation could serve
as the basis.

**Option 3: Avoid all CPython C API calls that take critical sections.**
Audit every CPython call in `pyProfileFn` and replace with lock-free
alternatives. `PyFrame_GetCode` also takes `Py_BEGIN_CRITICAL_SECTION(frame)`,
though frame lock contention is less likely since frames are per-thread.
