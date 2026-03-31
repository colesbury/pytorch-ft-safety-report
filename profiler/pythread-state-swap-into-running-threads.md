# `PyThreadState_Swap` into running threads during profiler init

- **Status:** FIXED (#178551)
- **Severity:** SEVERE
- **Component:** autograd/profiler_python.cpp

- **Shared state:** Other threads' `PyThreadState` and frame stacks.
- **Writer(s):** The profiler init code previously called `PyThreadState_Swap` to
  switch into each thread's state in order to walk its frame stack, while that
  thread was actively running.
- **Reader(s):** The target thread, actively executing Python bytecode and
  mutating its own frame stack.
- **Race scenario:** Thread A (main) starts the profiler and swaps into Thread B's
  `PyThreadState` to read its frame stack. Thread B is simultaneously running,
  pushing/popping frames. The frame pointers read by Thread A are concurrently
  mutated by Thread B, leading to use-after-free or corrupt stack walks.
- **Fix:** The current code uses `StopTheWorldGuard` (which calls
  `_PyEval_StopTheWorld` on 3.14t) to pause all threads before walking their
  stacks, and uses `PyThreadState_GetFrame` (which returns a strong reference)
  instead of `PyThreadState_Swap`.
