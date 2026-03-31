# Free-Threading Audit: distributed/autograd

## Architecture Summary

The distributed autograd subsystem extends PyTorch's autograd engine to
support automatic differentiation across RPC boundaries. Key components:

- **DistAutogradContainer** -- Singleton that manages the lifecycle of
  distributed autograd contexts. Sharded by context ID with per-shard
  mutexes. Each thread gets a `thread_local` current context ID.

- **DistAutogradContext** -- Stores per-pass state: send/recv autograd
  functions, accumulated gradients, the `GraphTask`, and outstanding RPC
  futures. Protected by a single internal `std::mutex lock_`.

- **DistEngine** -- Singleton that orchestrates the distributed backward
  pass. Maintains a set of initialized context IDs protected by
  `initializedContextIdsLock_`, and a global CPU thread for GPU-to-CPU
  continuations.

- **SendRpcBackward / RecvRpcBackward** -- Autograd `Node` subclasses
  that bridge the local autograd graph to the RPC transport.

Concurrency model: Distributed autograd is inherently multi-threaded.
RPC callbacks arrive on arbitrary threads. The `executeSendFunctionAsync`
path runs on RPC handler threads while `execute` (the main backward
entry) runs on the calling thread. Multiple gradient propagation RPCs
for the same context can arrive concurrently. Under the GIL, Python-level
serialization provided some implicit protection; under free-threading,
all C++ concurrency issues are fully exposed.

Note: This component uses `torch.distributed.rpc` which is deprecated
and not being developed for `torch.compile`. The concurrency tier system
from the audit guidelines does not directly apply. These issues would
only matter if distributed RPC is used under free-threading.

## SEVERE Issues

### 1. `addOutstandingRpc` callback captures raw `this` pointer -- use-after-free

- **Shared state:** `DistAutogradContext` instance (accessed via raw
  `this` in the lambda at `context.cpp:133`)
- **Writer(s):** `releaseContext` / `releaseContextIfPresent` erase the
  `shared_ptr<DistAutogradContext>` from the container's shard map
  (`container.cpp:188,206`). If this drops the last reference, the
  context is destroyed.
- **Reader(s):** The callback lambda registered in `addOutstandingRpc`
  (`context.cpp:133-148`) captures `this` as a raw pointer. When an
  outstanding RPC completes (possibly on an arbitrary RPC handler
  thread), the callback dereferences `this` to access `lock_` and
  `graphTask_`.
- **Race scenario:** Thread A calls `releaseContext(ctx_id)`, erasing
  the context from the shard map and dropping the last `shared_ptr`.
  The context is destroyed. Thread B's RPC handler thread completes
  an outstanding RPC and fires the callback, which accesses
  `this->lock_` and `this->graphTask_` on a freed object.
- **Severity:** SEVERE -- use-after-free, crash or memory corruption.
- **Suggested fix:** Capture a `shared_ptr<DistAutogradContext>` (or
  `weak_ptr` with a lock check) in the callback lambda instead of raw
  `this`. Alternatively, ensure `releaseContext` waits for all
  outstanding RPC callbacks to complete before dropping the context.

### 2. `SendRpcBackward::grads_` unprotected concurrent read/write with `retain_graph`

- **Shared state:** `SendRpcBackward::grads_` (a `variable_list` /
  `std::vector`) at `sendrpc_backward.h:30`
- **Writer(s):** `setGrads` is called from the RPC request handler
  (`request_callback_no_python.cpp:374`) on an RPC handler thread.
- **Reader(s):** `apply` reads and moves from `grads_` when the
  autograd engine executes this node (`sendrpc_backward.cpp:12,18`).
  `getGrads` is called from `executeSendFunctionAsync`
  (`dist_engine.cpp:470`) on the same thread as `setGrads`.
- **Race scenario:** In the normal case, `setGrads` happens-before
  `apply` due to the thread pool task queue synchronization. However
  with `retain_graph=true`, a second gradient propagation RPC for the
  same `autograd_message_id` can arrive while the first backward pass
  is still executing. Thread A's engine is in `apply` reading/moving
  `grads_` while Thread B's RPC handler calls `setGrads` on the same
  `SendRpcBackward` instance. The `std::vector` is concurrently written
  and read/moved, corrupting its internal pointers.
- **Severity:** SEVERE -- concurrent read/write on `std::vector` (only
  with `retain_graph=true`).
- **Suggested fix:** Add a mutex to `SendRpcBackward` protecting
  `grads_`, or copy the grads into a local variable in `apply` before
  the engine releases any synchronization.

## Significant Issues

### 3. `runGradCallbackForVariable` TOCTOU on gradient update

- **Shared state:** `accumulatedGrads_` (a `c10::Dict<Tensor,Tensor>`)
  at `context.h:135`
- **Writer(s):** `accumulateGrad` (called during backward execution)
  writes to `accumulatedGrads_` under `lock_`
  (`context.cpp:74-107`). `runGradCallbackForVariable` reads, releases
  the lock, calls the callback, re-acquires the lock, and writes back
  (`context.cpp:244-263`).
- **Reader(s):** Same as above -- both read and write the same dict
  entry.
- **Race scenario:** Thread A calls `runGradCallbackForVariable` for
  variable V. It reads grad G1 at line 250-254 and releases the lock.
  Thread B calls `accumulateGrad` for the same variable V, updating the
  gradient to G2 under the lock. Thread A's callback modifies its local
  copy of G1 and writes it back via `insert_or_assign` at line 260,
  overwriting G2. The gradient from Thread B is silently lost.
- **Severity:** Significant -- incorrect gradient values, silent
  numerical errors.
- **Suggested fix:** Hold the lock for the entire operation, or use a
  compare-and-swap pattern where the write-back checks that the gradient
  has not been modified since it was read.

### 4. `DistAutogradContainer::newAutogradMessageId` is not thread-safe

- **Shared state:** `next_autograd_message_id_` (declared as
  `std::atomic<int64_t>` at `container.h:156`) and `max_id_`
  (`container.h:159`)
- **Writer(s):** `newAutogradMessageId` increments
  `next_autograd_message_id_` via `operator++` (`container.cpp:104`).
- **Reader(s):** The `TORCH_INTERNAL_ASSERT` at line 103 reads
  `next_autograd_message_id_` and compares it to `max_id_`.
- **Race scenario:** Although the atomic increment itself is safe, the
  check-then-increment pattern (`TORCH_INTERNAL_ASSERT(val < max);
  return val++;`) is a TOCTOU. Thread A reads
  `next_autograd_message_id_` as `max_id_ - 1` (passes the assert),
  then Thread B also reads it as `max_id_ - 1` (also passes the
  assert). Both threads increment, resulting in
  `next_autograd_message_id_` exceeding `max_id_`. In practice the
  overflow is into the worker-id bits, causing a message ID that
  looks like it belongs to a different worker.
- **Severity:** Significant -- message ID collision across workers leads
  to incorrect autograd graph linkage and wrong gradients.
- **Suggested fix:** Use `fetch_add` and check the returned value:
  `auto id = next_autograd_message_id_.fetch_add(1);
  TORCH_INTERNAL_ASSERT(id < max_id_); return id;`

## Minor Issues

### 5. `DistAutogradContainer::newContext` overflow check is post-increment

- **Shared state:** `next_context_id_` (`std::atomic<int64_t>`,
  `container.h:141`) and `max_id_`
- **Writer(s):** `newContext` at `container.cpp:135` does
  `auto context_id = next_context_id_++;` (atomic, returns
  pre-increment value) then asserts `context_id < max_id_` at line 139.
- **Race scenario:** Unlike issue 4, the check here is on the consumed
  value (pre-increment), so each thread correctly validates its own ID.
  However, if the assert fires, the atomic has already been incremented
  past `max_id_`, meaning subsequent calls will also get invalid IDs
  until the process crashes. With 2^48 IDs per worker this is
  astronomically unlikely in practice.
- **Severity:** Minor -- the check is correct per-thread but
  non-recoverable on overflow; practically unreachable.
- **Suggested fix:** Check before increment using `fetch_add`, or
  accept the current behavior given the enormous ID space.

### 6. `DistAutogradContainer::numAutogradContexts` snapshot is non-atomic across shards

- **Shared state:** All shards' `contexts` maps
- **Reader(s):** `numAutogradContexts` iterates over all shards,
  locking each shard individually (`container.cpp:313-319`).
- **Writer(s):** `newContext`, `releaseContext`, etc. modify individual
  shards.
- **Race scenario:** Thread A creates a context (incrementing shard 3's
  count). Thread B calls `numAutogradContexts` which has already counted
  shard 3 but not shard 7. Thread A then releases a context in shard 7
  (decrementing its count). The total returned is inconsistent -- it
  reflects a state that never existed.
- **Severity:** Minor -- only used for debug info and logging.
- **Suggested fix:** Accept the inconsistency (it's only debug info),
  or acquire all shard locks before counting (expensive).

### 7. `DistAutogradContainer::initialized_` is a plain `bool` read without lock

- **Shared state:** `initialized_` (`bool`, `container.h:147`)
- **Writer(s):** `init` sets it to `true` under
  `dist_container_init_lock_` (`container.cpp:58`).
- **Reader(s):** `getInstance` reads it without any lock
  (`container.cpp:87`).
- **Race scenario:** Thread A calls `init`, setting `initialized_` to
  true. Thread B calls `getInstance` concurrently. Without a memory
  barrier, Thread B might see a stale `false` value even after `init`
  has completed. In practice, Python's import lock serializes module
  initialization, making this unlikely.
- **Severity:** Minor -- benign in practice due to Python import
  serialization; the only consequence is a spurious error message.
- **Suggested fix:** Make `initialized_` a `std::atomic<bool>`, or
  rely on the fact that `init` is called during module import which
  is serialized.
