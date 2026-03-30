# `CacheNode` destructor leak on interpreter shutdown

- **Status:** Open
- **Severity:** Minor
- **Tier:** Unknown
- **Component:** compiled_autograd
- **Source report:** [dynamo_compiled_autograd_v2.md](../dynamo_compiled_autograd_v2.md)

- **Tier:** N/A (shutdown-only)
- **Shared state:** `CacheNode::runtime_wrapper`, `CacheNode::compiled_fn`.
- **Race scenario:** `~CacheNode()` (lines 497-503) checks
  `Py_IsInitialized()` and leaks (`release()`) if the interpreter is
  finalizing. Under free-threading, a thread could still be executing compiled
  autograd when the interpreter starts finalizing. `Py_IsInitialized` could
  return true on entry to the destructor but become false by the time
  `Py_DECREF` runs, causing a crash.
- **Suggested fix:** Register an atexit handler to clear the cache before
  interpreter finalization.
