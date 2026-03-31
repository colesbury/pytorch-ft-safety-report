# Profiler PyThreadState_Swap into running threads (FIXED)

- **Status:** FIXED ([PR #178551](https://github.com/pytorch/pytorch/pull/178551))
- **Severity:** SEVERE
- **Component:** profiler_python.cpp

The profiler constructor previously used `PyThreadState_Swap` to switch
into other threads' state in order to walk their frame stacks and install
profile functions. Under free-threading, those threads are running
concurrently, so mutating their thread state from another thread caused
data races and crashes.

Fixed by using `StopTheWorldGuard` to pause all threads before accessing
their state, and `PyEval_SetProfileAllThreads` on Python 3.13+.
