# `_accessors` sort and `_fail_count` mutation unprotected when `check` is called from Python on non-root managers

- **Status:** Open
- **Severity:** Significant
- **Tier:** Tier 2
- **Component:** guards
- **Source report:** [dynamo_guards_v2.md](../dynamo_guards_v2.md)

- **Tier:** Tier 2
- **Shared state:** `GuardManager::_accessors` (vector), `GuardManager::_fail_count` (int64_t)
- **Writer(s):** `check_accessors_nopybind` (line 3537) sorts `_accessors` on failure. `check_leaf_guards_nopybind` (line 3485) and `check_accessors_nopybind` (line 3517) increment `_fail_count`.
- **Reader(s):** Same methods, plus `fail_count()` (line 3615) used in the sort comparator.
- **Race scenario:** Within a single root, the `_lock` mutex serializes access. But `GuardManager` exposes `check` to Python, and non-root `GuardManager` objects can be accessed from Python for debugging. If two threads call `check` on the same non-root `GuardManager` (or one calls it while guard evaluation is in progress on another thread), the concurrent sort of `_accessors` is undefined behavior (concurrent moves of `unique_ptr` elements). Additionally, `_fail_count++` is a non-atomic read-modify-write.
- **Consequence:** Corrupted `_accessors` vector (crash), or benign stale count.
- **Suggested fix:** Mark `_fail_count` as `std::atomic<int64_t>`. For `_accessors` sorting, the risk is low since Python-side debugging calls are rare, but documenting that `check` on non-root managers is not thread-safe, or routing all calls through the root lock, would be prudent.
