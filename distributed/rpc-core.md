# Free-Threading Audit: Distributed RPC Core

## Architecture Summary

The `torch.distributed.rpc` subsystem provides remote procedure call
functionality across workers. The core files audited are:

- **RpcAgent** (`rpc_agent.{h,cpp}`): Base class for RPC agents. Manages a
  singleton `currentRpcAgent_` (accessed via `std::atomic_load/store`), a retry
  map for failed RPCs (protected by `rpcRetryMutex_`), and configuration state.

- **TensorPipeAgent** (`tensorpipe_agent.{h,cpp}`): Concrete RPC agent using
  TensorPipe. Uses internal mutexes for `connectedPipes_`, `timeoutMap_`,
  `callCount`, `metrics`, and `networkData`. Uses `GroupMembershipLockGuard` for
  dynamic group membership changes.

- **RRefContext** (`rref_context.{h,cpp}`): Singleton managing RRef lifecycle.
  Tracks owner RRefs, pending users, forks, and confirmed users. Protected by
  `mutex_` for most maps. Uses `destroyedMutex_` for shutdown coordination.

- **RRef/UserRRef/OwnerRRef** (`rref_impl.{h,cpp}`): RRef implementations.
  UserRRef has `deletedOnOwnerMutex_` protecting deletion state. OwnerRRef uses
  a JitFuture for value synchronization.

- **PyRRef** (`py_rref.{h,cpp}`): Python wrapper around RRef. Holds
  `py::object` members (`type_`).

- **PythonRpcHandler** (`python_rpc_handler.{h,cpp}`): Singleton holding Python
  callable references for serialization/deserialization. Uses `init_lock_` for
  lazy initialization.

- **Message** (`message.{h,cpp}`): RPC message container. No shared mutable
  state.

- **Types** (`types.{h,cpp}`): Basic types (GloballyUniqueId, SerializedPyObj).
  `allowJitRRefPickle` is thread-local.

### Concurrency Model

The RPC subsystem has its own internal threading model independent of the
`torch.compile()` tiers described in the general audit guidelines. RPC uses
multiple threads inherently: TensorPipe I/O event loop threads, a thread pool
for processing requests, timeout polling threads, and retry threads. Multiple
Python threads may initiate RPC calls concurrently.

Under free-threading, the key new exposure is that Python threads calling into
the RPC C++ layer can now run truly concurrently, and Python objects
(`py::object`) manipulated by the RPC layer no longer have GIL-based
serialization. Additionally, callbacks that fire from internal RPC threads
(e.g., future completion callbacks) can race with Python thread operations on
shared state that was previously implicitly serialized by the GIL.

## SEVERE Issues

### 1. PythonRpcHandler py::object Members Accessed After cleanup() Without Synchronization

- **Shared state:** `pyRunFunction_`, `pySerialize_`, `pyDeserialize_`,
  `pyHandleException_`, `rrefProxyFunctions_`, `rrefTypeFunctions_` -- all
  `py::object` members of the singleton `PythonRpcHandler`.
- **Writer(s):** `PythonRpcHandler::init()` sets these (under `init_lock_`).
  `PythonRpcHandler::cleanup()` nullifies them (under `init_lock_`).
- **Reader(s):** `runPythonUdf()`, `serialize()`, `deserialize()`,
  `handleException()`, `isRemoteException()`, `getRRefProxyFunctions()`,
  `getRRefTypeFunctions()` -- none of these acquire `init_lock_`. They read the
  `py::object` members directly.
- **Race scenario:** Thread A calls `PythonRpcHandler::cleanup()` which calls
  `cleanupPyObj()`, setting the internal `PyObject*` pointers to null. Thread B
  is concurrently in `serialize()` and calls `pySerialize_(obj)`. Thread B reads
  a `py::object` whose `PyObject*` is being nullified by Thread A, leading to a
  null pointer dereference or use-after-free. Under the GIL, cleanup and usage
  could not overlap; under free-threading they can.
- **Severity:** SEVERE -- use-after-free / null pointer dereference.
- **Suggested fix:** Either hold `init_lock_` in every method that accesses the
  `py::object` members (expensive), or use an `std::atomic<bool>` initialized
  flag checked before each use, combined with ensuring cleanup waits for
  in-flight operations to complete (e.g., via a reference count or reader-writer
  lock).

### 2. PythonRpcHandler::init() Double-Init Race on py::object Members

- **Shared state:** `initialized_` flag and all `py::object` members.
- **Writer(s):** `PythonRpcHandler::init()` -- writes `py::object` members and
  then sets `initialized_ = true`, all under `init_lock_`.
- **Reader(s):** All public methods of `PythonRpcHandler` that read `py::object`
  members without acquiring `init_lock_`.
- **Race scenario:** Thread A calls `getInstance()` which calls `init()`. Thread
  A acquires `init_lock_`, starts writing `py::object` members. Thread B also
  calls `getInstance()`, blocks on `init_lock_` in `init()`, then proceeds.
  However, there is a subtler issue: `getInstance()` calls `init()` on every
  invocation. If `cleanup()` has been called (setting `initialized_ = false`),
  a subsequent `init()` call will re-initialize all `py::object` members. If
  another thread is concurrently reading these members (which doesn't acquire
  `init_lock_`), it will see partially-initialized state.
- **Severity:** SEVERE -- torn reads of `py::object` during re-initialization.
- **Suggested fix:** Same as issue 1. All readers must synchronize with writers.

### 3. RRef::handleError Static Local Map With Captured `this` Lambdas

- **Shared state:** `static std::unordered_map<RPCErrorType, ...> errorHandlers`
  inside `RRef::handleError()`.
- **Writer(s):** C++11 guarantees thread-safe initialization of function-local
  statics, so the initialization itself is safe. However, the lambdas stored in
  the map capture `this` -- a pointer to the current `RRef` instance. The static
  is initialized once with the first `RRef` that calls `handleError`, and all
  subsequent calls from *any* `RRef` instance will use the same lambdas with the
  stale `this` pointer from the first caller.
- **Reader(s):** Any RRef calling `handleError()`.
- **Race scenario:** RRef A calls `handleError()` first, initializing the static
  map with lambdas capturing `this = &A`. Later, RRef B calls `handleError()`.
  The static is already initialized, so B's call uses A's `this` pointer --
  calling `A->setTimedOut()` instead of `B->setTimedOut()`. This is a
  pre-existing bug independent of free-threading, but under free-threading A
  could be concurrently destroyed, making the captured `this` a dangling pointer.
- **Severity:** SEVERE -- dangling `this` pointer in static lambdas. This is
  actually a pre-existing bug that becomes more dangerous under free-threading
  because the original RRef can be destroyed while another thread uses the stale
  `this`.
- **Suggested fix:** Remove the `static` keyword so `errorHandlers` is
  constructed each time (it's small), or restructure to not capture `this` in the
  lambdas (e.g., pass the RRef as a parameter).

## Significant Issues

### 4. PyRRef::getRRefType() TOCTOU Race on type_ Cache

- **Shared state:** `PyRRef::type_` (an `std::optional<py::object>`).
- **Writer(s):** `getRRefType()` writes `type_` when `!type_.has_value()`.
- **Reader(s):** `getRRefType()` reads `type_.has_value()`. The destructor
  `~PyRRef()` also reads and decrefs `type_`.
- **Race scenario:** Two Python threads call `getRRefType()` on the same
  `PyRRef` concurrently. Both observe `!type_.has_value()`, both proceed to
  invoke the RPC to determine the type, both write to `type_`. The second write
  overwrites the first, leaking the reference count of the first `py::object`.
  Under the GIL, only one thread could be in `getRRefType()` at a time.
- **Severity:** Significant -- reference count leak leading to memory leak, or
  potential double-decref if the two writes race at a lower level.
- **Suggested fix:** Add a mutex to protect the `type_` cache, or use
  `std::call_once` to ensure the type is computed exactly once.

### 5. RpcAgent::typeResolver_ Unprotected Read/Write

- **Shared state:** `RpcAgent::typeResolver_` (`std::shared_ptr<TypeResolver>`).
- **Writer(s):** `setTypeResolver()` -- assigns via move.
- **Reader(s):** `getTypeResolver()` -- reads the shared_ptr.
- **Race scenario:** Thread A calls `setTypeResolver()` while Thread B calls
  `getTypeResolver()`. Since `std::shared_ptr` assignment is not atomic by
  default (only the control block operations are), a concurrent read and write
  can result in a torn read of the shared_ptr, leading to a crash. Under the
  GIL, these calls were serialized.
- **Severity:** Significant -- torn shared_ptr read can crash.
- **Suggested fix:** Use `std::atomic_load` / `std::atomic_store` for
  `typeResolver_` (same pattern already used for `currentRpcAgent_`), or protect
  with a mutex.

### 6. PythonRpcHandler Singleton: getInstance() Asserts No GIL Then Acquires GIL in init()

- **Shared state:** The `PythonRpcHandler` singleton's `py::object` members.
- **Writer(s):** `init()` acquires GIL via `PROFILE_GIL_SCOPED_ACQUIRE` and
  writes Python objects.
- **Reader(s):** `runPythonUdf()`, `serialize()`, `deserialize()` etc. also
  acquire GIL via `PROFILE_GIL_SCOPED_ACQUIRE`.
- **Race scenario:** Under free-threading, acquiring the GIL
  (`PyGILState_Ensure`) no longer provides mutual exclusion between threads. Two
  threads can both "hold the GIL" simultaneously. The `PROFILE_GIL_SCOPED_ACQUIRE`
  macro was being relied upon as an implicit lock for the `py::object` members.
  Under free-threading, Thread A in `init()` writing `pyRunFunction_` and Thread
  B in `runPythonUdf()` reading `pyRunFunction_` can proceed concurrently even
  though both have acquired the GIL.
- **Severity:** Significant -- this is exactly the "GIL as implicit lock"
  anti-pattern described in the audit guidelines. The C++ data (the `py::object`
  member variables) is not protected.
- **Suggested fix:** Use `init_lock_` in all methods, or a separate reader-writer
  lock for the `py::object` members.

### 7. RRefContext::destroyInstance() Accesses owners_ Without Lock

- **Shared state:** `RRefContext::owners_` and `pendingOwners_` maps.
- **Writer(s):** `destroyInstance()` iterates `owners_`, then calls
  `owners_.clear()` and `pendingOwners_.clear()` without holding `mutex_`.
- **Reader(s):** Other methods like `getOrCreateOwnerRRef()`,
  `getOwnerRRef()`, `delForkOfOwner()` access `owners_` under `mutex_`.
- **Race scenario:** Thread A calls `destroyInstance()` (during shutdown).
  Thread B is processing an incoming RPC message and calls
  `getOrCreateOwnerRRef()` which acquires `mutex_` and accesses `owners_`.
  Thread A's `owners_.clear()` without `mutex_` races with Thread B's locked
  access, corrupting the `unordered_map`.
- **Severity:** Significant -- map corruption during shutdown. Although shutdown
  is supposed to happen after all RPCs complete, in practice, in-flight callbacks
  or delayed messages can still be executing.
- **Suggested fix:** Hold `mutex_` when iterating and clearing `owners_` and
  `pendingOwners_` in `destroyInstance()`.

### 8. RRefContext::checkRRefLeaks() Accesses forks_ Without Lock

- **Shared state:** `RRefContext::forks_` map.
- **Writer(s):** `addForkOfOwner()`, `addForkOfOwnerIfNotPresent()`,
  `delForkOfOwner()`, `addSelfAsFork()` -- all under `mutex_`.
- **Reader(s):** `checkRRefLeaks()` iterates `forks_` without holding `mutex_`.
  Called from `destroyInstance()`.
- **Race scenario:** Same as issue 7. During shutdown, `checkRRefLeaks()`
  iterates `forks_` while another thread could still be modifying it under
  `mutex_` in a callback.
- **Severity:** Significant -- iterator invalidation / map corruption.
- **Suggested fix:** Hold `mutex_` when accessing `forks_` in
  `checkRRefLeaks()`.

## Minor Issues

### 9. TensorPipeAgent::getWorkerInfo() Returns Reference to Map Value After Releasing Lock

- **Shared state:** `workerNameToInfo_` and `workerIdToInfo_` maps.
- **Writer(s):** `updateGroupMembership()` modifies these maps under
  `GroupMembershipLockGuard`.
- **Reader(s):** `getWorkerInfo(const std::string&)` and
  `getWorkerInfo(worker_id_t)` find an iterator under the lock, release the
  lock, then dereference the iterator to return a `const WorkerInfo&`.
- **Race scenario:** Thread A calls `getWorkerInfo("worker1")`, obtains an
  iterator `it`, releases the lock. Thread B calls `updateGroupMembership()`
  with `isJoin=false`, erasing "worker1" from the map. Thread A dereferences
  `it`, which now points to erased memory.
- **Severity:** Minor -- only affects dynamic RPC groups (static groups don't
  erase from these maps at runtime), and `GroupMembershipLockGuard` only locks
  for static groups anyway (which seems like an existing design choice, possibly
  a pre-existing bug in the dynamic group path).
- **Suggested fix:** Return by value instead of by reference, or hold the lock
  longer and copy the result.

### 10. TensorPipeAgent::getWorkerInfos() Accesses workerNameToInfo_ Without Lock

- **Shared state:** `workerNameToInfo_` map.
- **Writer(s):** `updateGroupMembership()`.
- **Reader(s):** `getWorkerInfos()` iterates the map without any lock.
- **Race scenario:** Thread A calls `getWorkerInfos()` while Thread B is
  modifying the map in `updateGroupMembership()`. This causes iterator
  invalidation.
- **Severity:** Minor -- same dynamic group caveat as issue 9.
- **Suggested fix:** Acquire `GroupMembershipLockGuard` in `getWorkerInfos()`.

### 11. RRefContext::genGloballyUniqueId() Non-Atomic Increment of nextLocalId_

- **Shared state:** `RRefContext::nextLocalId_` (declared as
  `static std::atomic<local_id_t>`).
- **Writer(s):** `genGloballyUniqueId()` uses `nextLocalId_++`.
- **Reader(s):** Same.
- **Race scenario:** This is actually safe because `nextLocalId_` is already
  `std::atomic`. The `++` operator on atomics is atomic. No issue here.
- **Severity:** Not an issue -- included for completeness during audit.

### 12. PyRRef Destructor Races With getRRefType on type_

- **Shared state:** `PyRRef::type_` (`std::optional<py::object>`).
- **Writer(s):** `~PyRRef()` reads `type_.has_value()` and decrefs the
  contained `py::object`.
- **Reader(s):** `getRRefType()` checks `type_.has_value()` and potentially
  writes to `type_`.
- **Race scenario:** If a PyRRef is being destroyed on one thread while another
  thread is calling `getRRefType()`, the destructor may decref the `py::object`
  while `getRRefType()` is writing a new value to `type_`. This is a
  use-after-free on the `py::object`. Under the GIL, the destructor would be
  serialized with `getRRefType()`.
- **Severity:** Minor in practice -- typically a `PyRRef` shouldn't be accessed
  during destruction, but Python GC can trigger destructor at surprising times
  under free-threading (e.g., cyclic GC on a different thread).
- **Suggested fix:** Same as issue 4 -- add a mutex around `type_` access.

### 13. PythonRpcHandler::jitCompilationUnit() Returns Shared Pointer Without Synchronization

- **Shared state:** `PythonRpcHandler::jitCompilationUnit_`
  (`std::shared_ptr<torch::jit::CompilationUnit>`).
- **Writer(s):** `init()` sets it (under `init_lock_`), `cleanup()` nullifies
  it (under `init_lock_`).
- **Reader(s):** `jitCompilationUnit()` returns it without any lock.
  `PythonTypeResolver::resolveType()` calls
  `PythonRpcHandler::getInstance().jitCompilationUnit()`, also without lock.
- **Race scenario:** Thread A calls `cleanup()` which sets
  `jitCompilationUnit_ = nullptr` under `init_lock_`. Thread B concurrently
  calls `jitCompilationUnit()` without `init_lock_`, reads a partially-written
  `shared_ptr` (torn read).
- **Severity:** Minor -- only happens during cleanup/shutdown, but is still a
  data race on `shared_ptr`.
- **Suggested fix:** Use `std::atomic_load`/`std::atomic_store` or hold
  `init_lock_` when reading `jitCompilationUnit_`.

### 14. PythonRpcHandler::parseTypeFromStr() Reads typeParser_ Without Lock

- **Shared state:** `PythonRpcHandler::typeParser_`
  (`std::shared_ptr<jit::ScriptTypeParser>`).
- **Writer(s):** `init()` sets it (under `init_lock_`), `cleanup()` nullifies
  it (under `init_lock_`).
- **Reader(s):** `parseTypeFromStr()` dereferences it without any lock.
- **Race scenario:** Same pattern as issue 13. Torn read of `shared_ptr` if
  `cleanup()` runs concurrently.
- **Severity:** Minor -- same cleanup-time race.
- **Suggested fix:** Same as issue 13.
