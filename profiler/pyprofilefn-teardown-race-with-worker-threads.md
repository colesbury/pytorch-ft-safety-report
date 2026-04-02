# `pyProfileFn` teardown race: worker callbacks race with `disableProfiler`

- **Status:** Open
- **Severity:** SEVERE
- **Component:** autograd/profiler_python.cpp, autograd/profiler_kineto.cpp, profiler/collection.cpp
- **Confirmed by TSAN:** Yes (20 of 20 races in `tsan.log.105601`)

## Shared state

Per-thread `ThreadLocalSubqueue` objects owned by `RecordQueue::sub_queues_`,
and per-thread `ThreadLocalResults` objects owned by
`PythonTracer::thread_local_results_` (a `std::deque`). Both are read/destroyed
by the main thread during teardown while worker threads may still be writing to
them from inside `pyProfileFn`.

## Writers

Worker threads call `PythonTracer::pyProfileFn()` (profiler_python.cpp), which:
1. Calls `RecordQueue::getSubqueue()` (collection.cpp:722) to get or create a
   per-thread `ThreadLocalSubqueue`, then appends profiling events to it.
2. Accesses `ThreadLocalResults` fields (the `TraceKeyCacheState` tuple,
   `ValueCache`, exit times) to record call/return data.

## Readers / destroyers

The main thread calls `disableProfiler()` (profiler_kineto.cpp:874), which:
1. `state_ptr->removeCallback()` — removes torch op recording callback.
2. `finalizeTrace()` (profiler_kineto.cpp:454):
   a. `recordQueue.stop()` → `PythonTracer::stop()` — calls
      `setprofileAllThreads(nullptr)` (profiler_python.cpp:1164) to uninstall
      callbacks, but **does not wait for in-flight callbacks to finish**.
   b. `recordQueue.getRecords()` (collection.cpp) — iterates
      `sub_queues_` and reads each `ThreadLocalSubqueue`'s events.
   c. `python_tracer_->getEvents()` — post-processes `ThreadLocalResults`.
3. Eventually `PythonTracer` is destroyed, which destroys the
   `thread_local_results_` deque and all `ThreadLocalResults` objects.

## Race scenario

Thread A (main) calls `disableProfiler()`. Inside `finalizeTrace()`, it calls
`PythonTracer::stop()` which calls `setprofileAllThreads(nullptr)` to
uninstall the profiling callback. However, worker threads (T38, T40, T42, T43)
are already inside `pyProfileFn` — they entered before the callback was
uninstalled. `setprofileAllThreads` does not wait for in-flight callbacks.

Thread A then immediately proceeds to `RecordQueue::getRecords()`, which
iterates over `sub_queues_` and reads each subqueue's events. Simultaneously,
a worker thread is still in `pyProfileFn` writing to the same subqueue via
`getSubqueue()`. This is a data race on the subqueue contents.

After `getRecords()` returns, `PythonTracer` is eventually destroyed. Its
destructor destroys `thread_local_results_` (the deque of `ThreadLocalResults`).
A worker thread still in `pyProfileFn` holds a raw pointer to its
`ThreadLocalResults` and is accessing `TraceKeyCacheState` tuple members,
`ValueCache`, etc. This is a use-after-free.

## TSAN evidence

All 20 races in `tsan.log.105601` are this bug, in two variants:

**Variant 1 (6 races):** `RecordQueue::getRecords` on the main thread reads
subqueue data while `pyProfileFn` on a worker thread writes to the same
subqueue. The main thread holds `state_mutex_` (M0) but the worker doesn't
acquire it.

**Variant 2 (14 races):** `~PythonTracer` on the main thread destroys
`ThreadLocalResults` (via `std::_Destroy` / `~deque`) and `TraceKeyCacheState`
(via `~_Tuple_impl`) while worker threads are still reading these objects from
within `pyProfileFn`. Several of these manifest as races in `free()` — the
main thread frees memory the worker is still reading.

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

`PythonTracer::stop()` must ensure all in-flight `pyProfileFn` callbacks have
completed before returning. Options:

1. **Barrier after uninstall:** After `setprofileAllThreads(nullptr)`, use a
   mechanism to wait for all threads to exit any in-progress `pyProfileFn`
   call. An atomic per-thread "in callback" counter checked by the main thread
   would work.
2. **Epoch-based reclamation:** Defer destruction of `ThreadLocalResults` and
   subqueue data until all threads have acknowledged the stop.
3. **Stop-the-world:** Use `_PyEval_StopTheWorldAll` (available in 3.14t)
   around the teardown to ensure no thread is executing Python (and therefore
   no thread is in `pyProfileFn`).
