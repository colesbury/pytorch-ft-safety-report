# Global `std::function` feature flags in collection.cpp are not thread-safe

- **Status:** Open
- **Severity:** Significant
- **Tier:** Tier 1
- **Component:** profiler/collection.cpp

- **Shared state:** Four function-local static `std::function<bool()>` objects:
  `record_concrete_inputs_enabled_fn` (line 1621),
  `fwd_bwd_enabled_fn` (line 1640),
  `cuda_sync_enabled_fn` (line 1659),
  `record_tensor_addrs_enabled` (line 1678).
  Also `get_record_tensor_addrs_enabled` has a function-local
  `static std::optional<bool>` cache (line 1685).
- **Writer(s):** `set_*_fn()` and `set_*_val()` functions, called from Python
  to configure profiler behavior. These assign to the `std::function` object.
- **Reader(s):** `get_*()` functions, called from profiler callbacks on any
  thread during active profiling. For example,
  `get_record_concrete_inputs_enabled()` is called from
  `InputOutputEncoder::isSupportedScalarList` on every op.
- **Race scenario:** Thread A (Python main) reconfigures profiler settings via
  `set_record_concrete_inputs_enabled_fn(new_fn)`, which move-assigns to the
  `std::function`. Thread B is in a profiler callback calling
  `get_record_concrete_inputs_enabled()`, which calls the `std::function`'s
  `operator()`. `std::function` is not thread-safe; a concurrent read and
  write can corrupt its internal state (vtable pointer, stored callable),
  leading to a crash or calling through a corrupted function pointer.
- **Note:** These are effectively configuration flags. A stale read of the
  boolean *result* would be benign. But the race is on the `std::function`
  *object* itself, not on the boolean result, making it a real crash risk
  if a setter and getter overlap.
- **Suggested fix:** Replace with `std::atomic<bool>` for the common case. If
  the `std::function` overload is needed, protect with a `std::shared_mutex`
  or use an atomic pointer to a heap-allocated `std::function`.
