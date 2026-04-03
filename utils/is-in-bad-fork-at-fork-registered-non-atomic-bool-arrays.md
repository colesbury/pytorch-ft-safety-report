# `is_in_bad_fork` and `at_fork_registered` non-atomic bool arrays

- **Status:** FIXED — [pytorch/pytorch#178911](https://github.com/pytorch/pytorch/pull/178911)
- **Severity:** Minor
- **Component:** device_lazy_init
- **Source reports:** [utils__part1.md](../utils__part1.md)

- **Shared state:** `is_in_bad_fork` and `at_fork_registered` --
  file-scope static `std::array<bool, COMPILE_TIME_MAX_DEVICE_TYPES>`
  in `torch/csrc/utils/device_lazy_init.cpp` (lines 16-17).
- **Writer(s):**
  - `set_device_in_bad_fork()` (line 73-75): sets `is_in_bad_fork[i]`.
    Called from the `atfork` child handler (line 89) and from Python.
  - `register_fork_handler_for_device_init()` (line 80): sets
    `at_fork_registered[i] = true`. Called from Python before the first
    device runtime call.
- **Reader(s):**
  - `is_device_in_bad_fork()` (line 69-71): reads `is_in_bad_fork[i]`.
    Called from Python.
  - The `atfork` child handler (lines 83-96): reads
    `at_fork_registered[i]` to decide which devices to mark as bad fork.
- **Race scenario:** `is_in_bad_fork` is set in the atfork child handler
  and read from Python. In the child process after fork, only one thread
  exists, so the atfork handler itself is safe. But in the parent
  process, `set_device_in_bad_fork` could be called from Python while
  `is_device_in_bad_fork` is read from another thread. Similarly,
  `at_fork_registered` is set from Python and read from the atfork
  handler. Since `bool` writes are not formally atomic, these are
  technically data races, though on all mainstream platforms bool
  read/write is atomic in practice. The consequence of a stale read
  is benign: an extra lazy init attempt or a missed bad-fork warning.
- **Suggested fix:** Replace both arrays with
  `std::array<std::atomic<bool>, ...>` for formal correctness. The
  performance impact is negligible since these are not on hot paths.
