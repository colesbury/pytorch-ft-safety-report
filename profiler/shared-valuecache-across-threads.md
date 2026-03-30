# Shared `ValueCache` across threads

- **Status:** FIXED (#178552)
- **Severity:** SEVERE
- **Tier:** Tier 1
- **Component:** autograd/profiler_python.cpp

- **Shared state:** A single `ValueCache` instance containing multiple
  `ska::flat_hash_map` members, previously shared by all per-thread
  `ThreadLocalResults` via pointer.
- **Writer(s):** Every thread's `pyProfileFn` callback, which calls
  `TraceKeyCacheState::intern` -> `ValueCache::store`, inserting into the
  shared hash maps on every cache miss.
- **Reader(s):** Same threads during `ValueCache::load` in post-processing, and
  the store path itself (find-then-insert pattern).
- **Race scenario:** Thread A and Thread B both call profiled functions and hit
  cache misses simultaneously. Both call `ValueCache::store<CallType::PyCall>`
  which does `locations.find(key)` followed by `locations[key] = ...`. Two
  concurrent insertions into the same `ska::flat_hash_map` corrupt the map's
  internal state (rehash during insert, dangling bucket pointers, etc.),
  causing crashes.
- **Fix:** `ValueCache` is now a member of `ThreadLocalResults` (per-thread),
  so each thread writes to its own cache with no sharing.
