# `profiler_state_info_ptr` (shared_ptr) data race across threads

- **Status:** Open
- **Severity:** SEVERE
- **Tier:** Tier 1
- **Component:** autograd/profiler_kineto.cpp

- **Shared state:** `profiler_state_info_ptr` -- a file-scope
  `std::shared_ptr<ProfilerStateInfo>` at line 607.
- **Writer(s):**
  - `enableProfiler()` (line 847): assigns a new `shared_ptr` from the main
    thread.
  - `disableProfiler()` (line 876): assigns `nullptr` from the main thread.
- **Reader(s):**
  - `enableProfilerInChildThread()` (line 856): copies the `shared_ptr` from a
    child thread (`auto state_info_ptr = profiler_state_info_ptr`).
  - `isProfilerEnabledInMainThread()` (line 852): reads (compares to nullptr)
    from any thread.
  - `toggleTorchOpCollectionDynamic()` (line 707): dereferences
    `profiler_state_info_ptr->scopes` without null check.
- **Race scenario:** The main thread calls `disableProfiler()`, which sets
  `profiler_state_info_ptr = nullptr`. Concurrently, a child thread calls
  `enableProfilerInChildThread()`, which copies the `shared_ptr`. Copying a
  `std::shared_ptr` is NOT atomic with respect to concurrent writes to the
  same `shared_ptr` variable. The copy reads the control block pointer and
  object pointer non-atomically while the main thread is overwriting them.
  This is undefined behavior -- the control block's reference count can be
  corrupted, or the child thread can read a half-written pointer, leading to
  a crash or use-after-free.
- **Suggested fix:** Use `std::atomic<std::shared_ptr<ProfilerStateInfo>>`
  (C++20) or replace with a `std::mutex`-protected access pattern. For the
  simple null check in `isProfilerEnabledInMainThread`, an `atomic_load` would
  suffice.
