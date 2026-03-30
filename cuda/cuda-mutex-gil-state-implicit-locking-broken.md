# `cudaMutexGILState` implicit GIL-based deadlock prevention broken

- **Status:** Open
- **Severity:** Minor
- **Tier:** Tier 1
- **Component:** Module

- **Shared state:** `static PyGILState_STATE cudaMutexGILState` (Module.cpp line 495), and the invariant that the GIL is held while the CUDA free mutex is held.
- **Writer(s):**
  - `THCPModule_cudaLockMutex` (line 497): acquires the CUDA free mutex via `try_lock` in a busy loop, then calls `PyGILState_Ensure()` and stores the result.
- **Reader(s):**
  - `THCPModule_cudaUnlockMutex` (line 517): reads `cudaMutexGILState`, calls `PyGILState_Release()`, and unlocks the mutex.
- **Context:** The `cudaMutexGILState` variable itself is properly synchronized -- it is only written after acquiring the CUDA free mutex, and only read before releasing it. There is no data race on the variable.
- **Problem under free-threading:** The intent (documented in the comment on line 489) is to ensure that "a thread will NEVER lose the GIL as long as it holds the CUDA mutex". This prevents deadlocks where Thread A holds the CUDA mutex without the GIL, and Thread B holds the GIL and tries to acquire the CUDA mutex (e.g., during tensor deallocation deep within THC). Under free-threading, `PyGILState_Ensure` does not acquire any mutual exclusion lock -- it only attaches a thread state. The deadlock prevention invariant no longer holds, because there is no GIL to hold.
- **Consequence:** Potential deadlock in legacy code paths that rely on the CUDA free mutex + GIL invariant. In practice, these code paths (NCCL < 2 with GIL-based locking) are largely obsolete, so the practical risk is low.
- **Suggested fix:** Review whether `THCPModule_cudaLockMutex`/`cudaUnlockMutex` are still needed under free-threading. If so, the deadlock prevention needs to be rethought without relying on the GIL. The `PyGILState_Ensure`/`Release` calls can be removed or made conditional on `!Py_GIL_DISABLED`.
