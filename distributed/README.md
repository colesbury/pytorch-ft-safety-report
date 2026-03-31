# Free-Threading Audit: torch.distributed

Audit of `torch/csrc/distributed/` for Python 3.14t free-threading safety issues.
173 source files across three subsystems: c10d (collective comms), RPC, and
distributed autograd.

## Reports

| Report | Scope | SEVERE | Significant | Minor |
|--------|-------|--------|-------------|-------|
| [c10d-process-groups.md](c10d-process-groups.md) | ProcessGroupNCCL, Gloo, MPI, UCC, Wrapper | 2 | 5 | 5 |
| [c10d-stores-and-infra.md](c10d-stores-and-infra.md) | TCPStore, FileStore, logger, NCCLUtils, control plane | 0 | 3 | 3 |
| [c10d-bindings-and-ops.md](c10d-bindings-and-ops.md) | init.cpp, Ops, Functional, reducer, SymmetricMemory | 3 | 3 | 2 |
| [rpc-core.md](rpc-core.md) | tensorpipe_agent, rref_context, rref_impl, PythonRpcHandler | 3 | 5 | 4 |
| [rpc-bindings-and-utils.md](rpc-bindings-and-utils.md) | rpc/init.cpp, python_functions, request callbacks, profiler | 2 | 2 | 3 |
| [autograd.md](autograd.md) | dist_engine, context/container, send/recv backward | 2 | 2 | 3 |

Note: The RPC core and RPC bindings reports have overlapping findings for
`PythonRpcHandler` and `RpcAgent::typeResolver_` since both audits examined
these shared components. The deduplicated totals are below.

## Deduplicated Summary

### SEVERE (8 unique issues -- will crash or corrupt memory)

| # | Component | Issue | File |
|---|-----------|-------|------|
| 1 | c10d | `ProcessGroup::barrier()` static tensor reassigned on every call | ProcessGroup.hpp:796 |
| 2 | c10d | `groupRanks()` static vector written by `std::iota` on every call | ProcessGroupNCCL.cpp:2686, ProcessGroupGloo.cpp:728 |
| 3 | SymmetricMemory | `MemPoolAllocatorMap` has no mutex (unlike `AllocatorMap` above it) | SymmetricMemory.cpp |
| 4 | NVSHMEM | `get_rank_to_global_rank()` returns reference after releasing lock | NVSHMEMSymmetricMemory.cpp |
| 5 | NVSHMEM | `initialize_nvshmem_with_store()` TOCTOU on `static bool` | NVSHMEMSymmetricMemory.cpp |
| 6 | RPC | `PythonRpcHandler` py::object members read without lock, nullified by `cleanup()` | python_rpc_handler.cpp |
| 7 | RPC | `RRef::handleError` static map captures first caller's `this` -- dangling pointer | rref_impl.cpp |
| 8 | Autograd | `addOutstandingRpc` callback captures raw `this` -- use-after-free after context release | context.cpp:133 |

### Significant (12 unique issues -- incorrect behavior)

| # | Component | Issue |
|---|-----------|-------|
| 1 | c10d | `ProcessGroupNCCL::globalRank()` static initialized from nondeterministic first caller |
| 2 | c10d | `ProcessGroup::getBackend(DeviceType)` reads/writes `deviceTypeToBackend_` without lock |
| 3 | c10d | `ProcessGroupNCCL::isInitialized()` reads `devNCCLCommMap_` outside lock |
| 4 | c10d | `ProcessGroupNCCL::getCommSplitCounter()` iterates `devNCCLCommMap_` without lock |
| 5 | c10d | `ProcessGroupNCCL::shutdown()` iterates `devNCCLCommMap_` after releasing lock |
| 6 | c10d | `C10dLogger::registerLogger` -- `registered_` atomic but `logger_` unique_ptr is not |
| 7 | c10d | `Store::timeout_` mutated by `StoreTimeoutGuard` without synchronization |
| 8 | c10d | `thread_isolation_mode` non-atomic bool gates global vs per-thread registry |
| 9 | c10d | `get_group_info` returns dangling reference from `unordered_map` after releasing lock |
| 10 | RPC | `RpcAgent::typeResolver_` shared_ptr read/written without atomic ops |
| 11 | RPC | `RRefContext::destroyInstance()` accesses `owners_`/`forks_` without mutex |
| 12 | Autograd | `SendRpcBackward::grads_` unprotected concurrent read/write with `retain_graph` |
| 13 | Autograd | `runGradCallbackForVariable` releases lock, calls callback, re-acquires -- gradient loss |
| 14 | Autograd | `newAutogradMessageId` check-then-increment TOCTOU on atomic |

### Clean components (no issues found)

- **TCPStore** client/server: properly mutex-protected
- **FileStore**: mutex + flock
- **HashStore**: mutex on all ops
- **PrefixStore**: stateless delegate
- **Reducer** (DDP): well-synchronized with internal mutex
- **Ops.cpp, Functional.cpp**: stateless pass-throughs
- **control_plane::HandlerRegistry**: shared_mutex
- **NCCLComm**: recursive_mutex
- **quantization/**: pure compute, no global state

## Priority

The c10d issues (#1-2 barrier/groupRanks static races) are the most impactful
for near-term free-threading work since c10d is actively used. These are also
trivial to fix (remove `static`, use lambda-initialized magic static).

The RPC and distributed autograd issues (#6-8) are in deprecated code
(`torch.distributed.rpc`). They are real bugs but lower priority unless RPC
usage on 3.14t is planned.

The SymmetricMemory issues (#3-5) affect newer code that is likely to see
continued development and should be fixed.
