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
| [**dynamo**](dynamo/) | 8 SEVERE, 14 Significant, 10 Minor | Dict watcher races, ExtraState, compiled autograd; 3 fixed |
| [**profiler**](profiler/) | 2 SEVERE, 4 Significant, 4 Minor | profiler_kineto shared_ptr, GC callback UAF; 3 SEVERE fixed |
| [**autograd**](autograd/) | 1 Significant | functionToPyObject TOCTOU; 1 fixed |
| [**toplevel**](toplevel/) | 2 SEVERE, 1 Significant | DataLoader worker_pids, InternedStringsTable; 1 fixed |
| [**utils**](utils/) | 1 SEVERE | options_from_string; 5 fixed |
| [**cuda**](cuda/) | 1 Significant, 1 Minor | nccl communicators, CUDAPluggableAllocator shared_ptr; 1 fixed |
| [**distributed**](distributed/) | 7 SEVERE, 14 Significant, 18 Minor | groupRanks static race, SymmetricMemory missing mutex, PythonRpcHandler UAF, RRef dangling `this`; 1 fixed |
| [**inductor**](inductor/) | 5 SEVERE, 5 Significant, 6 Minor | aoti_kernel_cache_ zero synchronization, model_container partial locking, load_json_file static, runner registry |
| [**fx**](fx/) | 1 Minor | Lazy-init static PyObject* TOCTOU |
| [**tensor**](tensor/) | 0 | All state is write-once-during-init or config flags; broad-audit `default_backend` downgraded |

## Broad audit only (lower quality, not deep-audited)

| Component | Issues | Details |
|-----------|--------|---------|
| [jit](jit/) | 8 medium | Mostly pure C++ graph transforms |
| [jit_mobile](jit_mobile/) | 4 medium | Mostly pure C++ |
| [export](export/) | 1 medium | upgrader_registry |

## No issues found

`api/` (C++ frontend), `cpu/`, `functionalization/`, `functorch/`, `instruction_counter/`, `lazy/` (already mutex-protected), `monitor/`, `mps/`, `mtia/`, `multiprocessing/`, `onnx/`, `stable/`.
