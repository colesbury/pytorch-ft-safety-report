# PyTorch Free-Threading Safety Audit

Tracking thread-safety issues in `torch/csrc/` for Python 3.14t (free-threading / nogil).

See [CLAUDE.md](CLAUDE.md) for the audit methodology.

## Concurrency Model

Dynamo uses a tiered concurrency model (see [dynamo/](dynamo/)):

- **Tier 1 (urgent):** `torch.compile()` runs on the main thread; other threads do data loading. The main risk is dict watcher callbacks firing on data loader threads.
- **Tier 2 (goal):** Multiple threads call compiled functions concurrently.

## Summary by Component (deep-audited)

| Component | Issues | Details |
|-----------|--------|---------|
| [**dynamo**](dynamo/) | 11 SEVERE, 13 Significant, 11 Minor | Dict watcher races, ExtraState, compiled autograd |
| [**profiler**](profiler/) | 2 SEVERE, 4 Significant, 1 Minor | profiler_kineto shared_ptr, GC callback UAF; 2 SEVERE already fixed |
| [**autograd**](autograd/) | 2 Significant, 5 Minor | functionToPyObject TOCTOU, lazy-init patterns; 2 already fixed |
| [**toplevel**](toplevel/) | 3 SEVERE, 1 Significant, 2 Minor | DataLoader worker_pids, InternedStringsTable |
| [**utils**](utils/) | 3 SEVERE, 3 Significant, 1 Minor | device_lazy_init, python_dispatch maps |
| [**cuda**](cuda/) | 1 Significant, 2 Minor | nccl communicators, CUDAPluggableAllocator shared_ptr |
| [**distributed**](distributed/) | 8 SEVERE, 14 Significant, 16 Minor | barrier/groupRanks static races, SymmetricMemory missing mutex, PythonRpcHandler UAF, RRef dangling `this` |

## Broad audit only (lower quality, not deep-audited)

| Component | Issues | Details |
|-----------|--------|---------|
| [jit](jit/) | 8 medium | Mostly pure C++ graph transforms |
| [jit_mobile](jit_mobile/) | 4 medium | Mostly pure C++ |
| [fx](fx/) | 1 high, 2 medium | Lazy-init PyObject* |
| [export](export/) | 1 medium | upgrader_registry |
| [tensor](tensor/) | 1 medium | default_backend |

## No issues found

`api/` (C++ frontend), `cpu/`, `functionalization/`, `functorch/`, `inductor/` (low Python coupling), `instruction_counter/`, `lazy/` (already mutex-protected), `monitor/`, `mps/`, `mtia/`, `multiprocessing/`, `onnx/`, `stable/`.
