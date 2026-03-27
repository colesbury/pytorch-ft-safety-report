# Free-Threading Safety Audit: distributed_rpc_testing

## Overall Risk: Low

## Files Audited
- `distributed/rpc/testing/faulty_tensorpipe_agent.cpp`
- `distributed/rpc/testing/faulty_tensorpipe_agent.h`
- `distributed/rpc/testing/init.cpp`
- `distributed/rpc/testing/testing.h`

## Detailed Findings

### Issue 1 — FaultyTensorPipeAgent `failMessageCountMap_` guarded by mutex (Low)

**File:** `faulty_tensorpipe_agent.cpp`, lines 65-98 / `faulty_tensorpipe_agent.h`, lines 94-97

```cpp
std::unique_lock<std::mutex> lock(failMapMutex_);
auto it = failMessageCountMap_.find(key);
if (it == failMessageCountMap_.end()) {
  failMessageCountMap_[key] = 0;
}
if (failMessageCountMap_[key] < numFailSends_) {
  failMessageCountMap_[key]++;
  lock.unlock();
  // ... set error on future ...
} else {
  lock.unlock();
  return TensorPipeAgent::send(to, std::move(message), rpcTimeoutSeconds);
}
```

The `failMessageCountMap_` is properly guarded by `failMapMutex_`. The lock is released before making the actual send call or setting the error, avoiding holding the lock during potentially blocking operations. This is correct.

**Risk:** Low. Properly synchronized.

### Issue 2 — Static `msgMap` in `messageStringToType` (Low)

**File:** `faulty_tensorpipe_agent.cpp`, lines 129-141

```cpp
MessageType FaultyTensorPipeAgent::messageStringToType(
    const std::string& messageString) const {
  static std::unordered_map<std::string, MessageType> msgMap = {
      {"RREF_FORK_REQUEST", MessageType::RREF_FORK_REQUEST},
      ...
  };
  const auto& it = msgMap.find(messageString);
  ...
}
```

This is a static local map that is initialized once (thread-safe by C++ guarantees) and then only read. It is used in the constructor of `FaultyTensorPipeAgent` via `parseMessagesToFailInput` and `parseMessagesToDelay`, which are called during object construction.

**Risk:** Low. Read-only after thread-safe initialization.

### Issue 3 — Static `fromVecToString` helper (Low)

**File:** `faulty_tensorpipe_agent.cpp`, lines 8-9

```cpp
static std::string fromVecToString(const std::vector<char>& vec) {
  return std::string(vec.begin(), vec.end());
}
```

Pure function, no state.

**Risk:** Low.

### Issue 4 — Immutable configuration fields (Low)

**File:** `faulty_tensorpipe_agent.h`

```cpp
const int numFailSends_;
const std::vector<MessageType> messageTypesToFail_;
std::unordered_map<MessageType, float, std::hash<int>> messageTypesToDelay_;
```

`numFailSends_` and `messageTypesToFail_` are const after construction. `messageTypesToDelay_` is non-const but is only written during construction and then only read via `getDelayForMessage()` and the `pipeWrite` override. Since the agent is fully constructed before it starts processing, these reads are safe.

**Risk:** Low.

### Issue 5 — Testing module init `faulty_agent_init` (Low)

**File:** `testing/init.cpp`, lines 21-131

This function creates pybind11 module definitions and class bindings. Like the main `rpc_init`, it is called during Python module import, which is serialized by the import lock.

**Risk:** Low.

### Issue 6 — Static `methods[]` array (Low)

**File:** `testing/init.cpp`, lines 134-136

```cpp
static PyMethodDef methods[] = {
    {"_faulty_agent_init", faulty_agent_init, METH_NOARGS, nullptr},
    {nullptr, nullptr, 0, nullptr}};
```

Read-only static array after initialization.

**Risk:** Low.

### Issue 7 — testing.h header (Low)

**File:** `testing/testing.h`

Contains only a function declaration for `python_functions()`. No state.

**Risk:** Low.

## Summary

The testing subsystem is well-synchronized. The `FaultyTensorPipeAgent` class uses a mutex to protect its `failMessageCountMap_`, and all configuration fields are either const or only written during construction. The static message type map uses thread-safe static local initialization. The module init follows the standard pattern. No free-threading issues were found.
