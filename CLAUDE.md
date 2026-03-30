# How to Audit for Free-Threading (nogil) Thread-Safety Issues

This directory contains reports from auditing `torch/csrc/` for issues
related to Python 3.14t free-threading. The GIL has been removed, so C
extension code that relied on the GIL for thread safety is now exposed to
data races.

## What to look for

Focus on **shared mutable state and data flow**, not surface-level pattern
matching.

### High-value patterns (these cause real crashes)

1. **Shared mutable containers accessed from callbacks.** Dict watcher
   callbacks, weakref callbacks, and GC callbacks fire on whatever thread
   triggers them. If they read or write a global `std::unordered_map`,
   `std::vector`, or similar container that is also accessed during normal
   operation (e.g., guard evaluation), that's a data race on the container
   internals. This is the most dangerous pattern — it's not just a stale
   read, it's structural corruption of the container.

2. **Mutable state attached to shared objects.** Code objects are shared
   across all threads executing the same function. Any mutable state
   attached to a code object (e.g., `ExtraState`, cache entry lists) is
   concurrently accessible. Two threads calling the same function will
   race on this state.

3. **Cross-thread PyThreadState access.** Code that calls
   `PyThreadState_Swap` to switch into another thread's state, or walks
   another thread's frame stack, was safe under the GIL because the other
   thread was suspended. Under free-threading, the other thread is
   actively running — its frames and thread state are being mutated
   concurrently.

4. **Shared mutable caches across threads.** A single cache instance
   (e.g., `ValueCache` in the profiler) passed by pointer to per-thread
   callbacks means all threads write to the same maps/vectors
   concurrently.

### Low-value patterns (usually not worth reporting)

- **Configuration flags.** Non-atomic bools/ints that control behavior
  (feature flags set from Python) are not interesting. A stale read is
  benign — the worst case is one extra compilation or one skipped
  optimization.

- **Write-once-during-init globals.** Global `PyObject*` pointers set
  during module init and then read-only are safe in practice. Python's
  import lock serializes module initialization. Don't report these unless
  you can show they're accessed before init completes.

- **C++11 magic statics.** Function-local `static` variables have
  thread-safe initialization guaranteed by the C++ standard. The init is
  fine; only flag these if the *value* stored is a borrowed reference or
  the init calls into Python in a way that could deadlock.

- **`PyGILState_Ensure` / `PyGILState_Release`.** These still work under
  free-threading — you still need to call `PyGILState_Ensure` before
  calling Python C APIs from a non-Python thread. The only thing to watch
  out for is places where `PyGILState_Ensure` is used as an **implicit
  lock** to protect C/C++ data (e.g., "we hold the GIL so no other thread
  can touch this map"). That implicit locking no longer works. But the
  mere presence of `PyGILState_Ensure` is not a bug.

## Concurrency model

The goal is to make `torch.compile()` fully thread-safe — multiple
threads should be able to call compiled functions (and compile new ones)
concurrently. There are two tiers of priority:

### Tier 1 (urgent): Single-thread compile, multi-thread data loading

`torch.compile()` runs from the main thread only. Other threads are used
for data loading. Most dynamo internal state (ExtraState, cache entries,
compiled autograd) is only accessed from the main thread.

The main issue at this tier is **dict watcher callbacks**. Dynamo installs
`PyDict_Watch` on guarded dicts (module `__dict__`, function `globals()`,
etc.). These callbacks fire on whatever thread mutates the dict. A data
loader thread doing a lazy import or setting a module attribute can
trigger a watcher callback that races with guard evaluation on the main
thread.

### Tier 2 (goal): Full multi-thread torch.compile()

Multiple threads call compiled functions concurrently, triggering
concurrent guard evaluation, cache lookup, and potentially concurrent
compilation. This exposes all the ExtraState races (cache list corruption,
double-init, concurrent compilation mutating `frame_state`), the
compiled autograd races, and the guard manager shared state.

When auditing, note which tier an issue falls into.

## How to audit a file

For each piece of mutable state you find:

1. **Who writes it?** Trace all writers. Pay attention to callbacks
   (dict watchers, weakref callbacks, GC hooks) — these fire on arbitrary
   threads.

2. **Who reads it?** Trace all readers. Is the reader on the same thread
   as the writer, or can it be a different thread?

3. **Can the read and write happen concurrently?** Under free-threading,
   two Python threads can run simultaneously. If a writer is in a callback
   and a reader is on a hot path, they will race.

4. **What is the concrete consequence?** Construct a scenario: "Thread A
   is doing X (specific function/line), Thread B is doing Y, shared state
   Z is corrupted because..." If you can't construct a concrete scenario,
   the issue is probably not real.

5. **What breaks?** Is it a crash (corrupted container, use-after-free,
   null deref)? Incorrect behavior (wrong guard decision, wrong compiled
   code)? Or just a benign stale read?

## Report format

```markdown
# Free-Threading Audit: <component>

## Architecture Summary
Brief description of how the component works and its concurrency model.

## SEVERE Issues
Issues that will cause crashes or memory corruption.

## Significant Issues
Issues that cause incorrect behavior or are likely to cause problems.

## Minor Issues
Real but low-impact issues.

For each issue:
### Title
- **Shared state:** what is being raced on
- **Writer(s):** who writes it and when (include callback context)
- **Reader(s):** who reads it and when
- **Race scenario:** "Thread A does X while Thread B does Y → consequence"
- **Suggested fix:** brief
```

## Severity guide

- **SEVERE:** Concurrent access to a non-thread-safe container
  (`unordered_map`, `vector`, `list`), use-after-free, cross-thread
  PyThreadState manipulation. Will crash.

- **Significant:** TOCTOU on initialization, incorrect guard/compilation
  results from shared mutable state, reference counting races. Causes
  wrong behavior.

- **Minor:** Non-atomic flags that should be atomic for correctness but
  where a stale read is benign. Borrowed references in globals that are
  unlikely to be GC'd in practice.

- **Not worth reporting:** Config flags, write-once-during-init globals,
  C++11 magic statics with safe values.
