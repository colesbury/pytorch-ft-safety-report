# `LogAPIUsageOnceFromPython` static `std::unordered_set` concurrent access

- **Status:** Open
- **Severity:** SEVERE
- **Tier:** Tier 1
- **Component:** Module.cpp

- **Shared state:** `seen` -- a `static std::unordered_set<std::string>` inside
  `LogAPIUsageOnceFromPython()` (Module.cpp:2127).
- **Writer(s):**
  - `LogAPIUsageOnceFromPython()` calls `seen.insert(event)` when the event
    has not been seen before. This is called from Python via
    `torch._C._log_api_usage_once()`.
- **Reader(s):**
  - `LogAPIUsageOnceFromPython()` calls `seen.count(event)` to check if the
    event is already present. Same function, but can be called from any thread.
- **Race scenario:** Thread A and Thread B both call
  `torch._C._log_api_usage_once("some.event")` concurrently. Thread A calls
  `seen.count(event)`, finds it absent, and proceeds to `seen.insert(event)`.
  Meanwhile Thread B also calls `seen.count(event)`, which reads the hash map
  internals while Thread A is inserting. `std::unordered_set::insert` can
  rehash the container, invalidating internal bucket pointers. Thread B
  traversing the bucket chain during rehash reads freed or partially
  initialized memory. This is undefined behavior on the container internals
  and can crash.
  The comment at line 2125 says "Guaranteed to be invoked from Python under
  GIL, no locking on map needed" -- this is exactly the GIL-removal pattern
  that breaks under free-threading.
- **Suggested fix:** Use a `std::mutex` to protect the `seen` set, or switch to
  a concurrent set. Alternatively, use `c10::call_once` per event string
  (impractical for dynamic strings), or use a thread-safe concurrent hash set.
  A simple `std::mutex` around the check-and-insert is sufficient since this
  is not a hot path.
