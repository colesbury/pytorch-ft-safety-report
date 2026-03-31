# `device_lazy_init` TOCTOU race on `is_initialized` arrays

- **Status:** Fix pending — [pytorch/pytorch#178911](https://github.com/pytorch/pytorch/pull/178911)
- **Severity:** Significant
- **Component:** device_lazy_init
- **Source reports:** [utils__part1.md](../utils__part1.md)

- **Shared state:** `is_initialized` -- a file-scope static
  `std::array<bool, COMPILE_TIME_MAX_DEVICE_TYPES>` in
  `torch/csrc/utils/device_lazy_init.cpp` (line 15). The code comment
  on line 29 explicitly says "Protected by the GIL."
- **Writer(s):**
  - `device_lazy_init()` (line 62): sets `is_initialized[i] = true`
    after calling `_lazy_init`.
  - `set_requires_device_init()` (line 65-67): sets
    `is_initialized[i] = !value`, called from Python and from the
    `atfork` child handler.
- **Reader(s):**
  - `is_device_initialized()` (line 22-25): reads `is_initialized[i]`.
    Called from `device_lazy_init()` as the guard check (line 33), and
    also exposed to Python.
  - `device_lazy_init()` itself (line 33): calls `is_device_initialized`
    as a guard, then proceeds to do expensive work if it returns false.
- **Race scenario:** Thread A calls `device_lazy_init(CUDA)` and reads
  `is_initialized[CUDA] == false` at line 33. Thread B concurrently
  calls `device_lazy_init(CUDA)` and also reads `false`. Both threads
  proceed to call `torch.cuda._lazy_init()`, causing double
  initialization. The `pybind11::gil_scoped_acquire` on line 28 does
  not provide mutual exclusion under free-threading -- the GIL is a
  no-op.
  Additionally, `set_requires_device_init` (called from Python or
  from the atfork handler) writes to `is_initialized[i]` without
  acquiring any lock, racing with readers in `is_device_initialized`.
  Since `bool` is not guaranteed atomic on all platforms, this is
  formally undefined behavior.
- **Suggested fix:** Replace the `is_initialized` array with
  `std::array<std::atomic<bool>, ...>` and use `c10::call_once`
  with per-device `c10::once_flag` entries to guard the actual
  initialization call. The existing comment about ASAN + call_once
  deadlocks may need revisiting, but the TOCTOU must be fixed.
  Alternatively, use a per-device `std::mutex` to protect the
  check-then-init sequence.
