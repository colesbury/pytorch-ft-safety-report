# Free-Threading Safety Audit: distributed_rpc__part2

## Overall Risk: Medium

## Files Audited
- `distributed/rpc/python_rpc_handler.cpp`
- `distributed/rpc/python_rpc_handler.h`
- `distributed/rpc/request_callback.cpp`
- `distributed/rpc/request_callback.h`
- `distributed/rpc/request_callback_impl.cpp`
- `distributed/rpc/request_callback_impl.h`
- `distributed/rpc/request_callback_no_python.cpp`
- `distributed/rpc/request_callback_no_python.h`
- `distributed/rpc/rpc.h`
- `distributed/rpc/rpc_agent.cpp`
- `distributed/rpc/rpc_agent.h`
- `distributed/rpc/rpc_command_base.h`
- `distributed/rpc/rref_context.cpp`
- `distributed/rpc/rref_context.h`
- `distributed/rpc/rref_impl.cpp`

## Detailed Findings

### Issue 1 — PythonRpcHandler singleton lazy initialization (Medium)

**File:** `python_rpc_handler.cpp`, lines 68-131

```cpp
void PythonRpcHandler::init() {
  std::lock_guard<std::mutex> guard(init_lock_);
  if (!initialized_) {
    PROFILE_GIL_SCOPED_ACQUIRE;
    // ... imports and initializations ...
    initialized_ = true;
  }
}

PythonRpcHandler& PythonRpcHandler::getInstance() {
  TORCH_INTERNAL_ASSERT(!PyGILState_Check());
  static PythonRpcHandler* handler = new PythonRpcHandler();
  handler->init();
  return *handler;
}
```

The singleton uses a leaky pointer + `init()` with a mutex-guarded `initialized_` flag. The C++ static local guarantees thread-safe construction. The `init()` method uses a mutex, which is correct. However, `initialized_` is a plain `bool` that is read without the lock in `init()` (actually it is read under the lock since the check is inside the lock guard scope). After `init()` completes, the member fields like `pyRunFunction_`, `pySerialize_`, etc. are `py::object` instances that are then accessed by various methods without the init lock. These reads are safe as long as `cleanup()` is not called concurrently, which is governed by the RPC lifecycle.

The `TORCH_INTERNAL_ASSERT(!PyGILState_Check())` assertion will fire incorrectly under free-threading where `PyGILState_Check()` always returns 1.

**Risk:** Medium. The singleton init pattern is sound, but the `!PyGILState_Check()` assertion is broken under free-threading.

### Issue 2 — PythonRpcHandler `cleanup()` races with ongoing usage (Medium)

**File:** `python_rpc_handler.cpp`, lines 99-115

```cpp
void PythonRpcHandler::cleanup() {
  std::lock_guard<std::mutex> guard(init_lock_);
  PROFILE_GIL_SCOPED_ACQUIRE;
  cleanupPyObj(pyRunFunction_);
  // ...
  initialized_ = false;
}
```

The cleanup acquires the init_lock and GIL, then destroys the py::objects. However, concurrent callers of `serialize()`, `deserialize()`, or `runPythonUdf()` do not hold `init_lock_` -- they acquire the GIL via `PROFILE_GIL_SCOPED_ACQUIRE` but rely on the py::objects being valid. Under free-threading, a concurrent `serialize()` call could be using `pySerialize_` while `cleanup()` is destroying it. Under the GIL this was partially serialized (both paths acquire GIL), but under free-threading the GIL no longer provides mutual exclusion.

**Risk:** Medium. This is a lifecycle issue -- `cleanup()` is called during `rpc.join()` shutdown, and the RPC framework should ensure no RPCs are in-flight. But the lack of formal synchronization means a late-arriving RPC callback could race.

### Issue 3 — PROFILE_GIL_SCOPED_ACQUIRE macro (Low)

**File:** `python_rpc_handler.cpp`, lines 13-25

The macro calls `RpcAgent::getCurrentRpcAgent()->isGILProfilingEnabled()` before acquiring the GIL, then records timing. Under free-threading this is fine as `isGILProfilingEnabled()` reads an atomic bool and `getCurrentRpcAgent()` uses `std::atomic_load`.

**Risk:** Low.

### Issue 4 — RpcAgent static `currentRpcAgent_` (Low)

**File:** `rpc_agent.cpp`, lines 256-290

```cpp
std::shared_ptr<RpcAgent> RpcAgent::currentRpcAgent_ = nullptr;

bool RpcAgent::isCurrentRpcAgentSet() {
  return std::atomic_load(&currentRpcAgent_) != nullptr;
}

std::shared_ptr<RpcAgent> RpcAgent::getCurrentRpcAgent() {
  std::shared_ptr<RpcAgent> agent = std::atomic_load(&currentRpcAgent_);
  ...
}

void RpcAgent::setCurrentRpcAgent(std::shared_ptr<RpcAgent> rpcAgent) {
  // Uses std::atomic_compare_exchange_strong / std::atomic_exchange
  ...
}
```

All accesses to the static `currentRpcAgent_` use `std::atomic_load`, `std::atomic_exchange`, and `std::atomic_compare_exchange_strong`, which provide correct thread safety under free-threading.

**Risk:** Low. Properly synchronized.

### Issue 5 — RpcAgent::typeResolver_ not synchronized (Low-Medium)

**File:** `rpc_agent.cpp`, lines 292-298

```cpp
void RpcAgent::setTypeResolver(std::shared_ptr<TypeResolver> typeResolver) {
  typeResolver_ = std::move(typeResolver);
}

std::shared_ptr<TypeResolver> RpcAgent::getTypeResolver() {
  TORCH_INTERNAL_ASSERT(typeResolver_, "Type resolver is not set!");
  return typeResolver_;
}
```

The `typeResolver_` field is a `std::shared_ptr` that is set once during agent initialization and read during message deserialization. The read and write are not formally synchronized. Under free-threading, if `setTypeResolver` and `getTypeResolver` could overlap, this would be a data race on a `shared_ptr`. In practice, `setTypeResolver` is called once during `_set_and_start_rpc_agent` before the agent starts processing, so this is likely safe in practice.

**Risk:** Low-Medium. Safe by convention but not by construction.

### Issue 6 — RRefContext singleton and `destroyed_` flag (Low)

**File:** `rref_context.cpp`, lines 81-105

```cpp
RRefContext& RRefContext::getInstance() {
  static RRefContext* context = new RRefContext(RpcAgent::getCurrentRpcAgent());
  return *context;
}
```

Leaky singleton with C++ static local. Thread-safe construction. The `destroyed_` flag is guarded by `destroyedMutex_`, and `mutex_` protects the internal maps. This is correctly synchronized.

**Risk:** Low.

### Issue 7 — RRef::handleError uses static local map with `this` capture (Medium)

**File:** `rref_impl.cpp`, lines 67-82

```cpp
void RRef::handleError(RPCErrorType errorType, const JitFuture& jitFuture) {
  static std::unordered_map<
      RPCErrorType,
      std::function<void(const JitFuture& jitFuture)>,
      std::hash<int>>
      errorHandlers = {
          {RPCErrorType::TIMEOUT,
           [this](const JitFuture& /* unused */) { setTimedOut(); }},
          ...
      };
  errorHandlers.find(errorType)->second(jitFuture);
}
```

This is a static local map initialized with lambdas that capture `this`. The static local initialization is thread-safe, but the lambdas capture `this` from the first call's object instance. Subsequent calls on different `RRef` instances will still use the first caller's `this`. This is a pre-existing bug that is independent of free-threading.

**Risk:** Medium. Pre-existing bug -- the static local captures `this` from the first invocation, making the error handlers incorrect for all subsequent RRef instances.

### Issue 8 — RegisterWorkerInfoOnce static registration (Low)

**File:** `rpc_agent.cpp`, lines 6-13

```cpp
RegisterWorkerInfoOnce::RegisterWorkerInfoOnce() {
  static auto workerInfo = torch::class_<WorkerInfo>("dist_rpc", "WorkerInfo")
                               .def(torch::init<std::string, int64_t>());
}
```

Thread-safe static local initialization. Safe under free-threading.

**Risk:** Low.

### Issue 9 — RRefContext::nextLocalId_ atomic counter (Low)

**File:** `rref_impl.cpp`, line 36 / `rref_context.h`, line 241

```cpp
static std::atomic<local_id_t> RRefContext::nextLocalId_{0};
```

Used via `nextLocalId_++` which is an atomic increment. Safe.

**Risk:** Low.

## Summary

The primary concerns are around the `PythonRpcHandler` singleton: the `!PyGILState_Check()` assertion will fail under free-threading, and the `cleanup()` method could race with ongoing usage if the RPC lifecycle is not strictly enforced. The `RRef::handleError` static map with captured `this` is a pre-existing bug. The `RpcAgent` static agent pointer and `RRefContext` are properly synchronized with atomic operations and mutexes respectively.
