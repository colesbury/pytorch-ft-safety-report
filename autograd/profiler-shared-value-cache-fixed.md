# Profiler shared ValueCache across threads (FIXED)

- **Status:** FIXED ([PR #178552](https://github.com/pytorch/pytorch/pull/178552))
- **Severity:** SEVERE
- **Component:** profiler_python.cpp

The profiler previously used a single shared `ValueCache` instance for all
threads. All profile callbacks from all threads wrote into the same
`ska::flat_hash_map` instances without synchronization, causing container
corruption and crashes.

Fixed by making `ValueCache` a per-thread member of `ThreadLocalResults`
rather than a shared instance on the `PythonTracer`.
