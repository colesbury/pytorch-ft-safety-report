# Free-Threading Audit: RPC Bindings, Utilities, and Profiling

## Architecture Summary

The distributed RPC subsystem provides Python bindings (`init.cpp`), utility
functions for serialization/deserialization (`utils.cpp`), Python-facing RPC
call wrappers (`python_functions.cpp`), a request callback pipeline
(`request_callback*.cpp`), a server-side process-global profiler
(`profiler/server_process_global_profiler.cpp`), a remote profiler manager
(`profiler/remote_profiler_manager.cpp`), and supporting infrastructure
(agent utils, tensorpipe utils, testing faulty agent).

**Concurrency model:** The RPC framework is inherently multi-threaded. The
`TensorPipeAgent` uses a thread pool to process incoming requests. Multiple
worker threads execute `RequestCallback::processMessage()` concurrently.
Callbacks on `JitFuture` objects can fire on arbitrary threads. The server
profiler is explicitly designed for cross-thread use: a global profiler
state is read by all RPC worker threads and results are pushed from those
threads into a shared container.

Under the GIL, Python object manipulation in callbacks was implicitly
serialized. Under free-threading, callbacks that touch Python objects or
shared C++ state can race with each other and with the main thread.

Note: The RPC subsystem is slated for deprecation, so the practical risk
here is limited to users who continue to use it on Python 3.14t. Many of
the issues below already had partial mitigation via explicit mutexes. The
most relevant issues are those where the GIL was being used as an implicit
lock for C++ data.

## SEVERE Issues

### 1. `RpcAgent::typeResolver_` written and read without synchronization

- **Shared state:** `RpcAgent::typeResolver_` (`std::shared_ptr<TypeResolver>`)
- **Writer(s):** `RpcAgent::setTypeResolver()` called from
  `_set_and_start_rpc_agent` in `init.cpp` during RPC initialization.
- **Reader(s):** `RpcAgent::getTypeResolver()` called from many
  deserialization paths (`ScriptCall::fromMessage`, `ScriptResp::fromMessage`,
  `ScriptRemoteCall::fromMessage`, `PythonRemoteCall::fromMessage`,
  `readWrappedPayload` in `utils.cpp`, `RRefProto`) on RPC worker threads.
- **Race scenario:** Thread A (main thread) calls `_set_and_start_rpc_agent`,
  which calls `setTypeResolver()` and then `rpcAgent->start()`. If `start()`
  returns quickly and an RPC arrives before the `shared_ptr` write is visible
  to a worker thread, that worker thread calls `getTypeResolver()` and reads a
  partially-written `shared_ptr`. Under free-threading, `std::shared_ptr`
  read/write from different threads without synchronization is undefined
  behavior (control block corruption, use-after-free).
- **Severity:** SEVERE -- concurrent non-atomic read/write of `shared_ptr`
  causes UB. In practice the window is narrow because `setTypeResolver` is
  called before `start()`, but the lack of any memory barrier means worker
  threads spawned by `start()` have no guarantee of seeing the write.
- **Suggested fix:** Make `typeResolver_` an `std::atomic<std::shared_ptr<TypeResolver>>`
  (C++20), or protect it with a mutex, or set it before the agent is started
  and add an acquire/release fence.

### 2. `PythonRpcHandler` singleton fields accessed from multiple threads with GIL as implicit lock

- **Shared state:** `PythonRpcHandler`'s member fields: `pyRunFunction_`,
  `pySerialize_`, `pyDeserialize_`, `pyHandleException_`,
  `rrefProxyFunctions_`, `rrefTypeFunctions_`, `jitCompilationUnit_`,
  `typeParser_`, `initialized_`.
- **Writer(s):** `PythonRpcHandler::init()` (guarded by `init_lock_` mutex)
  and `PythonRpcHandler::cleanup()` (also guarded by `init_lock_`).
- **Reader(s):** `runPythonUdf()`, `serialize()`, `deserialize()`,
  `handleException()`, `isRemoteException()`, `parseTypeFromStr()`,
  `jitCompilationUnit()`, `getRRefProxyFunctions()`, `getRRefTypeFunctions()`
  -- all called from RPC worker threads.
- **Race scenario:** Thread A calls `cleanup()` which acquires `init_lock_`,
  sets `pyRunFunction_` to None, and sets `initialized_ = false`. Thread B
  is concurrently in `runPythonUdf()` (no lock held) and accesses
  `pyRunFunction_`. The `py::object` is being mutated (dec_ref + ptr = nullptr)
  while Thread B tries to call it. This is a use-after-free on the Python
  object. Under the GIL this was safe because `cleanup()` and `runPythonUdf()`
  could not execute simultaneously. Under free-threading, they can.
- **Severity:** SEVERE -- use-after-free / null deref on `py::object`.
- **Suggested fix:** The reader methods should also acquire `init_lock_`
  (or a separate `std::shared_mutex` for read-side concurrency), or
  `cleanup()` should be redesigned to not null out function objects while
  they may still be in use (e.g., use reference counting or a shutdown flag
  that readers check under the lock).

## Significant Issues

### 3. `PyRRef::type_` cache has TOCTOU race

- **Shared state:** `PyRRef::type_` (`std::optional<py::object>`)
- **Writer(s):** `PyRRef::getRRefType()` sets `type_` on first call.
- **Reader(s):** Subsequent calls to `getRRefType()` read the cached value.
  Also read in the destructor `~PyRRef()`.
- **Race scenario:** If a `PyRRef` is shared across threads (e.g., stored in
  a Python object accessible from multiple threads), two threads could call
  `getRRefType()` simultaneously. Both see `!type_.has_value()`, both make the
  RPC to fetch the type, and both write to `type_`. The second write overwrites
  the first `py::object` without decref-ing it (memory leak), and the
  `py::object` assignment itself is not thread-safe (the `optional`'s internal
  state could be corrupted).
- **Severity:** Significant -- memory leak or corrupted `py::object` if
  `PyRRef` is shared across threads.
- **Suggested fix:** Protect `type_` with a mutex or use `std::call_once` to
  ensure single initialization.

### 4. Server process-global profiler `pushResultRecursive` races with `popRange`

- **Shared state:** The `StateStackEntry` linked list accessed via
  `currentStateStackEntryPtr` (global). The `prevPtr_` and `statePtr_` fields
  of `StateStackEntry` are `const`, so traversal of the linked list is safe.
  However, `pushResultRecursive` follows the `prevPtr_` chain while
  `popRange()` modifies `currentStateStackEntryPtr`.
- **Writer(s):** `pushRange()` and `popRange()` write `currentStateStackEntryPtr`
  under `wLockType wlock(currentStateStackEntryMutex)`.
- **Reader(s):** `pushResultRecursive()` traverses the list via `prevPtr()`
  without holding the lock. It receives the head pointer by value (from
  `StateStackEntry::current()` which does hold the read lock), so the
  traversal is on a snapshot. However, while traversal is in progress,
  `popRange()` can remove the top entry and allow it to be destroyed (if
  its refcount drops to zero). The `shared_ptr` copy in `current()` keeps
  the head alive, but `pushResult()` on an already-popped entry writes
  results into a `State` that may have already been consumed by
  `disableServer()`.
- **Severity:** Significant -- results pushed after `disableServer()` swaps
  out `results_` are silently lost, producing incorrect profiling data.
  Not a crash because the `State::resultsMutex_` protects the container.
- **Suggested fix:** Accept that results pushed after `disableServer()` are
  lost, or hold the read lock for the duration of result pushing.

## Minor Issues

### 5. `RpcAgent::profilingEnabled_` is atomic, but the GIL-profiling macro is non-atomic in spirit

- **Shared state:** `RpcAgent::profilingEnabled_` (already `std::atomic<bool>`)
- **Writer(s):** `enableGILProfiling(bool flag)` from Python.
- **Reader(s):** `PROFILE_GIL_SCOPED_ACQUIRE` macro in
  `python_rpc_handler.cpp`.
- **Race scenario:** The flag itself is atomic, so there's no data race.
  However, `PROFILE_GIL_SCOPED_ACQUIRE` reads `shouldProfileGIL` and
  then conditionally records timing. If the flag changes between the read
  and the timing measurement, the timing is skipped or spuriously recorded.
  This is benign -- at worst one timing sample is missed or extra.
- **Severity:** Minor -- benign stale read.
- **Suggested fix:** No fix needed; the current atomic is sufficient.

### 6. `barrierId` global atomic increment is not sequentially consistent with store operations

- **Shared state:** `static std::atomic<int> barrierId(0)` in `agent_utils.cpp`.
- **Writer(s):** `getNextKeyIds()` increments it.
- **Reader(s):** `syncCallCount()` calls `getNextKeyIds()`.
- **Race scenario:** If two threads call `syncCallCount()` concurrently
  (not typical -- this is called during join), the atomic increment
  ensures uniqueness. But the `barrierId++` followed by `barrierId.load()`
  is a TOCTOU: another thread could increment between the two operations,
  causing both threads to use the same or a skipped ID.
- **Severity:** Minor -- `syncCallCount` is called in a coordinated shutdown
  path, unlikely to be called concurrently from multiple threads.
- **Suggested fix:** Use `barrierId.fetch_add(1)` and use the returned value
  directly instead of a separate `.load()`.

### 7. `FaultyTensorPipeAgent::messageStringToType` lazy static map initialization

- **Shared state:** `static std::unordered_map<std::string, MessageType> msgMap`
  in `messageStringToType()`.
- **Writer(s):** C++ runtime initializes on first call (thread-safe by C++11).
- **Reader(s):** Subsequent calls read from the map.
- **Race scenario:** The map is initialized with an initializer list (C++11
  magic static, thread-safe). After initialization, the map is read-only.
  No real issue.
- **Severity:** Minor -- not a real issue. C++11 guarantees thread-safe
  initialization, and the map is never mutated after construction.
- **Suggested fix:** None needed.
