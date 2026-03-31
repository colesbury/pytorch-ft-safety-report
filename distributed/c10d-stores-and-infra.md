# Free-Threading Audit: c10d Stores and Infrastructure

## Architecture Summary

The c10d distributed package provides key-value stores for distributed
rendezvous and synchronization, plus supporting infrastructure (logging,
NCCL utilities, control plane).

**Store hierarchy:** `Store` (abstract base) is subclassed by `TCPStore`,
`FileStore`, `HashStore`, and `PrefixStore` (a wrapper that delegates to
another store). `TCPStore` uses a client-server architecture where the
server runs on a background thread (`TCPStoreMasterDaemon` or
`LibUVStoreDaemon`). The client side serializes operations via
`activeOpLock_`.

**Concurrency model:** Stores are typically created once per process group
and accessed from a single Python thread, but under free-threading
multiple Python threads may share and concurrently call into the same
store instance. The TCPStore server backends are inherently single-threaded
(poll loop or libuv event loop), so server-side state is safe. The
concern is client-side races and shared global mutable state.

**Key components:**
- `TCPStore` client: per-instance `activeOpLock_` protects client socket
- `FileStore`: per-instance `activeFileOpLock_` protects file ops
- `HashStore`: per-instance mutex `m_` protects all operations
- `PrefixStore`: stateless wrapper, delegates to underlying store
- `Store` base: `timeout_` field is unprotected
- `C10dLogger`: global singleton with registration race
- `socket.cpp`: non-atomic global `disable_getnameinfo` flag and
  function-local `lastLog` static
- `NCCLUtils`: non-atomic `nccl_nonblocking_timeout` lazy init,
  static `ncclDataType` map in header
- `control_plane`: `HandlerRegistry` properly uses `shared_mutex`

## Significant Issues

### 1. C10dLogger::registerLogger race condition

- **Shared state:** `C10dLogger::logger_` (`std::unique_ptr<C10dLogger>`)
  and `C10dLogger::registered_` (`std::atomic<bool>`)
- **Writer(s):** `C10dLogger::registerLogger()` writes `logger_` and
  `registered_` with no lock. The `registered_` atomic guards the
  early-return check, but there is no synchronization between the store
  to `registered_` and the move to `logger_`.
- **Reader(s):** `C10dLogger::getLogger()` reads `registered_` then
  dereferences `logger_.get()`.
- **Race scenario:** Thread A calls `registerLogger()` and stores
  `registered_ = true` before `logger_ = std::move(logger)` completes
  (the compiler/CPU could reorder these). Thread B calls `getLogger()`,
  sees `registered_ == true`, and reads `logger_.get()` which is still
  null or in a partially-moved state. Result: null pointer dereference or
  reading an incomplete object.
  Even if the compiler happens to order them correctly, `std::unique_ptr`
  is not atomic -- a concurrent reader could observe a torn pointer.
- **Severity:** Significant -- could cause a null dereference crash, though
  in practice `registerLogger` is typically called once during setup.
- **Suggested fix:** Use `std::call_once` for registration, or protect
  both `logger_` and `registered_` with a single mutex. Alternatively,
  store `registered_` with `std::memory_order_release` and load with
  `std::memory_order_acquire`, and ensure the `logger_` assignment
  happens-before the `registered_` store.

### 2. Store::timeout_ unprotected concurrent read/write

- **Shared state:** `Store::timeout_`
  (`std::chrono::milliseconds`), a protected member of the base class.
- **Writer(s):** `Store::setTimeout()` writes `timeout_` with no
  synchronization. `StoreTimeoutGuard` calls `setTimeout` in its
  constructor and destructor.
- **Reader(s):** `Store::getTimeout()` reads `timeout_`.
  Every store operation that uses `timeout_` (e.g., `TCPStore::wait`,
  `FileStore::get`, `HashStore::get`) reads the field.
- **Race scenario:** Thread A creates a `StoreTimeoutGuard` which calls
  `setTimeout(newTimeout)`. Concurrently, Thread B calls `get()` which
  reads `timeout_`. Since `std::chrono::milliseconds` is 8 bytes, on most
  architectures the read is atomic in practice, but this is technically
  undefined behavior. More concretely, if Thread A's `StoreTimeoutGuard`
  destructor restores the old timeout while Thread B is mid-operation,
  Thread B could use a timeout value that was only intended for Thread A's
  scope.
- **Severity:** Significant -- `StoreTimeoutGuard` is fundamentally
  thread-unsafe. Two threads using `StoreTimeoutGuard` on the same store
  will corrupt each other's timeout values (TOCTOU on save/restore).
- **Suggested fix:** Make `StoreTimeoutGuard` use thread-local timeout
  overrides, or add a mutex around timeout access. If timeouts are
  per-operation, pass them as parameters instead of mutating shared state.

### 3. nccl_nonblocking_timeout lazy initialization race

- **Shared state:** Function-local `static int timeout = -2` in
  `nccl_nonblocking_timeout()`.
- **Writer(s):** First caller that sees `timeout == -2` writes the
  computed value. However, this is not C++11 magic static -- it's a
  `static` local initialized to `-2` (the initialization is thread-safe),
  but the subsequent `if (timeout == -2)` check-and-write is a plain
  non-atomic read-modify-write.
- **Reader(s):** All subsequent callers read `timeout`.
- **Race scenario:** Thread A enters the function, reads `timeout == -2`,
  begins computing the value from the environment variable. Thread B
  enters concurrently, also reads `timeout == -2` (the write hasn't
  happened yet), and also begins computing. Both threads then write to
  `timeout`. Since `int` writes are typically atomic on most architectures,
  the result is likely correct (both compute the same value), but this is
  technically a data race (undefined behavior per C++ standard).
- **Severity:** Significant (UB) -- in practice benign since both threads
  compute the same value, but it is a real data race on a non-atomic
  variable.
- **Suggested fix:** Use a proper C++11 magic static with a lambda:
  `static int timeout = []() { ... }();`

## Minor Issues

### 4. socket.cpp disable_getnameinfo non-atomic flag

- **Shared state:** `static bool disable_getnameinfo` in
  `formatSockAddr()`.
- **Writer(s):** Set to `true` when `getnameinfo` fails.
- **Reader(s):** Read on every call to `formatSockAddr()`.
- **Race scenario:** Thread A calls `formatSockAddr()`, `getnameinfo`
  fails, sets `disable_getnameinfo = true`. Thread B concurrently calls
  `formatSockAddr()`, reads the flag. Non-atomic bool read/write is
  technically UB, but a stale read is benign -- the worst case is one
  extra DNS resolution attempt.
- **Severity:** Minor -- benign stale read, but technically UB.
- **Suggested fix:** Make it `static std::atomic<bool>`.

### 5. socket.cpp lastLog non-atomic static in retry loop

- **Shared state:** `static auto lastLog` in
  `SocketConnectOp::tryConnect(int family)`.
- **Writer(s):** Updated with `lastLog = now` inside the retry loop.
- **Reader(s):** Read to check `(now - lastLog) >= 30s`.
- **Race scenario:** Multiple threads connecting to the same address
  concurrently will race on reading/writing this `time_point`. The
  consequence is merely extra or suppressed log messages.
- **Severity:** Minor -- benign, affects only log throttling.
- **Suggested fix:** Make it `static std::atomic<...>` or accept the
  benign race with a comment.

### 6. ncclDataType static map defined in header

- **Shared state:** `static std::map<at::ScalarType, ncclDataType_t>
  ncclDataType` defined in `NCCLUtils.hpp`.
- **Reader(s):** `getNcclDataType()` does `ncclDataType.find(type)`.
- **Writer(s):** This is a `static` variable in a header, meaning each
  translation unit gets its own copy (internal linkage). The map is
  initialized at static init time and never modified afterward.
- **Race scenario:** Since each TU has its own copy and the map is
  read-only after initialization, there is no actual data race. However,
  using `static` in a header wastes memory with one copy per TU.
- **Severity:** Minor -- no thread-safety issue, but poor practice
  (should be `inline` or `const`).
- **Suggested fix:** Change to `static const std::map` or move to a .cpp
  file. Not a thread-safety issue, just a code quality observation.

## Not Reported (Examined and Found Safe)

**TCPStore client-side operations:** All public methods acquire
`activeOpLock_` before accessing `client_`, providing proper
synchronization for multi-thread use.

**TCPStore server (TCPStoreMasterDaemon, LibUVStoreDaemon):** The server
runs on a single background thread with a poll/libuv event loop. All
mutable state (`tcpStore_`, `waitingSockets_`, `keysAwaited_`) is only
accessed from that single thread. Safe.

**TCPServer::cachedServers_:** Protected by `TCPServer::cache_mutex_`.
Safe.

**FileStore:** All operations acquire `activeFileOpLock_` plus
filesystem-level `flock()`. Safe for multi-thread access within a process.

**HashStore:** All operations acquire `m_` mutex. Safe.

**PrefixStore:** Stateless wrapper, delegates all operations to the
underlying store which provides its own synchronization. Safe.

**control_plane::HandlerRegistry:** Uses `std::shared_mutex` properly.
Safe.

**WaitCounterHandler:** Uses `c10::Synchronized` for the counter map and
`std::atomic` for counter data. Safe.

**NCCLComm:** Uses `recursive_mutex` (`mutex_`) to protect internal state.
The `getAsyncError`, `getNcclComm`, `abort`, `finalize`, etc. all acquire
the lock. Safe.

**BackgroundThread::is_running_:** `std::atomic<bool>`, safe.

**getNcclVersion/getNcclVersionTuple/getNcclVersionNumber:** Use proper
C++11 magic statics with lambdas. Thread-safe initialization.

**Socket::initialize (Winsock):** Uses C++11 magic static. Safe.
