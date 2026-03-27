# Free-Threading Safety Audit: distributed_autograd_functions

## Overall Risk: Low

## Files Audited
- `distributed/autograd/functions/recvrpc_backward.cpp`
- `distributed/autograd/functions/recvrpc_backward.h`
- `distributed/autograd/functions/sendrpc_backward.cpp`
- `distributed/autograd/functions/sendrpc_backward.h`

## Findings

### 1. `RecvRpcBackward` holds `weak_ptr` to autograd context -- No Issue
**File:** `distributed/autograd/functions/recvrpc_backward.h`
**Lines:** 31-35

```cpp
std::weak_ptr<DistAutogradContext> autogradContext_;
```

The `weak_ptr` is set in the constructor and subsequently only read (via `lock()`) in `apply()`. The `lock()` operation on `weak_ptr` is thread-safe. The resulting `shared_ptr` keeps the context alive for the duration of `apply()`. All subsequent operations on the context (`retrieveGraphTask`, `addOutstandingRpc`) are protected by the context's internal mutex.

### 2. `RecvRpcBackward::apply` -- No Issue
**File:** `distributed/autograd/functions/recvrpc_backward.cpp`
**Lines:** 22-63

This function:
1. Reads from `input_metadata()` -- inherited from `Node`, set during construction and immutable afterward.
2. Locks the weak context pointer (line 34).
3. Calls `sharedContext->retrieveGraphTask()` and `sharedContext->addOutstandingRpc()` -- both are mutex-protected.
4. Calls `rpc::RpcAgent::getCurrentRpcAgent()` -- returns a shared_ptr from a global that is set during RPC initialization.

No shared mutable state is accessed without synchronization. Safe.

### 3. `SendRpcBackward::grads_` member -- Low Risk
**File:** `distributed/autograd/functions/sendrpc_backward.cpp` and `sendrpc_backward.h`

The `grads_` member variable is set via `setGrads()` and read via `getGrads()` and `apply()`. There is no mutex protecting `grads_`.

However, in the distributed autograd protocol, `setGrads()` is called when gradients arrive over the wire, before the function is enqueued for execution. The autograd engine then calls `apply()` which reads and moves from `grads_`. The sequencing is enforced by the autograd engine's task queue (the node is only executed after it is enqueued with its inputs set). This constitutes a happens-before relationship, so no data race occurs in practice.

### 4. Immutable fields -- No Issue
**Files:** `recvrpc_backward.h`, `sendrpc_backward.h`

- `RecvRpcBackward::autogradMetadata_` (const)
- `RecvRpcBackward::fromWorkerId_` (const value type)
- `RecvRpcBackward::deviceMap_` (const)

All are set in the constructor and never modified. Safe.

## Summary

These autograd function classes (`RecvRpcBackward`, `SendRpcBackward`) serve as nodes in the distributed autograd graph. They have minimal shared mutable state. `RecvRpcBackward` safely uses a weak_ptr to access the autograd context under its mutex. `SendRpcBackward`'s `grads_` field relies on the autograd engine's execution ordering for synchronization. No free-threading issues were identified.
