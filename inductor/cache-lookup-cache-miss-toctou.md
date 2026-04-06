# TOCTOU between `cache_lookup` and `cache_miss`

- **Status:** Open
- **Severity:** Significant
- **Component:** aoti_eager/kernel_holder.cpp

- **Shared state:** `aoti_kernel_cache_` vector
- **Issue:** `cache_lookup()` returns false (no match). `cache_miss()` assumes
  this still holds. No lock spans the check-then-insert sequence.
- **Race scenario:** Thread A misses cache. Thread B concurrently inserts a
  matching entry. Thread A compiles and inserts a duplicate.
- **Consequence:** Wasted compilation, duplicate entries. Combined with the
  [concurrent read/write issue](aoti-kernel-cache-no-synchronization.md), the
  concurrent insertions are also UB.
- **Suggested fix:** Hold lock across lookup-then-insert, or re-check under
  write lock before inserting.
