# `leaked_python_filenames_` vector concurrent push_back invalidates pointers

- **Status:** Open
- **Severity:** SEVERE
- **Component:** python_dispatch
- **Source reports:** [utils__part2.md](../utils__part2.md)

- **Shared state:** `leaked_python_filenames_` -- a file-scope static
  `std::vector<std::string>` in `torch/csrc/utils/python_dispatch.cpp`
  (line 34).
- **Writer(s):**
  - `_dispatch_library` lambda (line 515-516): calls
    `leaked_python_filenames_.emplace_back(file)` then immediately reads
    `leaked_python_filenames_.back().c_str()`.
  - `_dispatch_clear_leaked_python_filenames` lambda (line 537): calls
    `leaked_python_filenames_.clear()`.
- **Reader(s):**
  - Any `torch::Library` object that was given a `leaked_file` pointer
    from a previous `_dispatch_library` call. These Library objects store
    the raw `const char*` and use it for debug/error messages throughout
    their lifetime.
- **Race scenario:** Thread A calls `_dispatch_library`, which does
  `emplace_back(file)` and then `back().c_str()`. Thread B concurrently
  calls `_dispatch_library`, which does another `emplace_back`. If B's
  `emplace_back` triggers a reallocation of the vector's internal buffer,
  all existing `c_str()` pointers are invalidated -- the string objects
  are moved to a new buffer, and the old buffer (containing the strings
  whose `c_str()` was returned to A and to all prior Library objects) is
  freed. Thread A now holds a dangling `const char*` pointer.
  Additionally, concurrent `emplace_back` calls on `std::vector` are
  undefined behavior even without reallocation.
  The `clear()` function is even worse: it destroys all strings while
  Library objects may still hold raw pointers into them.
- **Suggested fix:** Replace the `std::vector` with a data structure that
  does not invalidate existing elements on insertion (e.g., `std::deque`
  or `std::list`), and protect it with a mutex. Alternatively, use
  a `folly::ConcurrentHashMap` or similar concurrent container.
  The `clear()` function should be removed or gated behind a check
  that no Library objects are alive.
