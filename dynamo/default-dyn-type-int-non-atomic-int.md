# `default_dyn_type_int` non-atomic int

- **Status:** Open
- **Severity:** Minor
- **Tier:** Tier 1
- **Component:** compiled_autograd
- **Source report:** [dynamo_compiled_autograd_v2.md](../dynamo_compiled_autograd_v2.md)

- **Tier:** Tier 1 (written by any thread calling `set_autograd_compiler`,
  read during backward)
- **Shared state:** `default_dyn_type_int` (file-static `int`, line 55)
- **Writer(s):** `set_autograd_compiler()` (line 1263)
- **Reader(s):** `get_default_dyn_type()` (line 883) during
  `_compiled_autograd_impl`.
- **Race scenario:** Thread A is in `_compiled_autograd_impl` calling
  `get_default_dyn_type()` while Thread B calls `set_autograd_compiler`. The
  read/write is on a plain `int` with no synchronization -- a C++ data race
  (undefined behavior). In practice the consequence is benign (wrong dynamic
  type for one compilation), but it is technically UB.
- **Suggested fix:** Use `std::atomic<int>`.
