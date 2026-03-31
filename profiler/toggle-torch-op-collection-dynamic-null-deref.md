# `toggleTorchOpCollectionDynamic` dereferences `profiler_state_info_ptr` without null check

- **Status:** Open
- **Severity:** Significant
- **Component:** autograd/profiler_kineto.cpp

- **Shared state:** `profiler_state_info_ptr` -- the same global
  `std::shared_ptr<ProfilerStateInfo>` described in the shared_ptr data race
  issue.
- **Writer(s):** `disableProfiler()` sets it to `nullptr`.
- **Reader(s):** `toggleTorchOpCollectionDynamic()` (line 707) reads
  `profiler_state_info_ptr->scopes` unconditionally when `enable` is true.
- **Race scenario:** The profiler is active. A thread calls
  `toggleTorchOpCollectionDynamic(true)`, which checks `state_ptr` (the
  TLS profiler state) but does NOT check whether `profiler_state_info_ptr`
  is still valid. Concurrently, the main thread calls `disableProfiler()`
  which sets `profiler_state_info_ptr = nullptr`. The toggle function
  dereferences a null (or half-written) `shared_ptr`, causing a segfault.
  Even without the shared_ptr atomicity issue, this is a TOCTOU bug.
- **Suggested fix:** Copy `profiler_state_info_ptr` into a local before
  dereferencing, and check for null. Combined with the atomic shared_ptr
  fix from the parent issue.
