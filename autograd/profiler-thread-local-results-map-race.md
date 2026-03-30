# Profiler thread_local_results_map_ concurrent read/write

- **Status:** Open
- **Severity:** Minor
- **Tier:** Tier 2
- **Component:** profiler_python.cpp

## Shared state

```cpp
std::unordered_map<PyThreadState*, ThreadLocalResults*> thread_local_results_map_;
```

This map is a member of `PythonTracer`. It maps thread states to their
per-thread profiling results.

## Writers

The `PythonTracer` constructor populates this map (line 1057) inside a
`StopTheWorldGuard` block. However, `restart()` calls
`setprofileAllThreads()` which installs the profile callback on all threads
-- including any new threads spawned since the initial construction. Those
new threads are NOT in `thread_local_results_map_` and the map is never
updated for them.

While the constructor's writes are safe (stop-the-world), the fundamental
problem is that `thread_local_results_map_` is never updated after
construction. New threads spawned during profiling will call
`pyProfileFn`, which calls `findThreadLocalResults()`, which reads the map.
If another profiling operation were to modify the map (e.g., in a
hypothetical "add new thread" path), it would race.

## Readers

`findThreadLocalResults()` (line 963) reads the map from within
`pyProfileFn`, which runs on arbitrary threads without synchronization.

## Race scenario

Currently, the map is written only during construction (under STW) and read
afterwards, so there is no concurrent write+read race in the current code.
However, this is fragile: `restart()` does not re-enumerate threads, which
means if a new thread calls `pyProfileFn`, it calls
`findThreadLocalResults()` which returns `nullptr`, and the callback returns
early. This is safe but means new threads are silently not profiled after
`restart()`.

The actual race occurs if `interpreterThreads()` is called (which walks the
thread list) concurrently with a new Python thread being created/destroyed
-- but this is done under GIL or STW in the current code.

## Assessment

This is currently safe by construction (write under STW, then read-only),
but the design is fragile. Marking as SEVERE because if anyone adds a path
that modifies the map after construction, the existing unsynchronized reads
would immediately become a data race.

## Suggested fix

Use a concurrent map, or accept the current design and add a prominent
comment that `thread_local_results_map_` must not be modified after
construction.
