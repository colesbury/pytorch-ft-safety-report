# `aoti_kernel_cache_` has zero synchronization

- **Status:** Open
- **Severity:** SEVERE
- **Component:** aoti_eager/kernel_holder.cpp

`AOTIPythonKernelHolder` is a dispatcher kernel functor — one instance per
(op, dispatch key) pair, shared across all threads. Its `aoti_kernel_cache_`
(`std::vector<AOTIKernelMetadata>`, `kernel_holder.h:77`) is read and written
with no locks, no atomics, nothing.

- **Shared state:** `aoti_kernel_cache_` vector
- **Writer(s):** `cache_miss()` at `kernel_holder.cpp:472` calls `push_back()`,
  which may reallocate the vector's internal buffer. The compilation window is
  very long (calls into Python), making concurrent `cache_miss()` calls likely.
- **Reader(s):** `cache_lookup()` at `kernel_holder.cpp:213-218` iterates the
  vector on every op dispatch.
- **Race scenarios:**
  1. **Read/write race (use-after-free):** Thread A enters `cache_miss()` and
     calls `push_back`, triggering reallocation. Thread B is concurrently in
     `cache_lookup()` iterating the vector — it dereferences the freed buffer.
  2. **Write/write race:** Two threads both miss the cache, both compile kernels,
     both call `push_back` concurrently. Two concurrent `push_back` on
     `std::vector` is UB — corrupted size/capacity, double-free.
  3. **TOCTOU (cache_lookup → cache_miss):** No lock spans the
     check-then-insert sequence, so duplicate entries are silently created.
- **Tier:** 2 (multi-thread inference), but also 1 if data loader threads
  dispatch the same AOTI op.
- **Suggested fix:** Add `std::shared_mutex` to `AOTIPythonKernelHolder`.
  Shared lock in `cache_lookup`, exclusive lock in `cache_miss` with re-check
  before insert.
