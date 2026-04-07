# `CacheNode::check_dynamic_sizes` mutates shared state without external protection

- **Status:** Open
- **Severity:** Significant
- **Tier:** Tier 2
- **Component:** compiled_autograd
- **Source report:** [dynamo_compiled_autograd_v2.md](../dynamo_compiled_autograd_v2.md)

- **Tier:** Tier 2 (requires concurrent backwards, currently blocked by mutex)
- **Shared state:** `CacheNode::expected_sizes`, `runtime_wrapper`,
  `compiled_fn`.
- **Writer(s):** `check_dynamic_sizes()` (line 509) modifies `expected_sizes`
  and may null out `runtime_wrapper`/`compiled_fn` on a shape-change cache
  miss.
- **Reader(s):** `compiled_autograd()` reads `cache->runtime_wrapper` and
  `cache->compiled_fn` after the check. `is_cache_empty()` reads
  `compiled_fn` and `next.empty()`.
- **Race scenario:** Currently the compiled autograd mutex serializes backward
  calls, so two threads cannot be in `check_dynamic_sizes` on the same
  `CacheNode` simultaneously. However, `is_cache_empty()` (line 643) has no
  synchronization and reads `compiled_fn` and `next` without any lock. If
  Thread A is inside `check_dynamic_sizes` nulling out `compiled_fn` (line
  567-568) while Thread B calls `is_cache_empty`, the read of `compiled_fn`
  races with the write. This is a data race on the `THPObjectPtr` internals.
- **Suggested fix:** `is_cache_empty` should acquire the compiled autograd
  mutex.
