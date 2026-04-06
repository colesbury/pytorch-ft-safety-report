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
| [**profiler**](profiler/) | 3 SEVERE, 4 Significant, 4 Minor | pyProfileFn teardown race (TSAN-confirmed), profiler_kineto shared_ptr, GC callback UAF; 2 SEVERE already fixed |
| [**autograd**](autograd/) | 1 Significant, 2 Minor | functionToPyObject TOCTOU, lazy-init patterns |
| [**toplevel**](toplevel/) | 2 SEVERE, 1 Significant, 2 Minor | DataLoader worker_pids, InternedStringsTable; 1 fixed |
| [**utils**](utils/) | 1 SEVERE | options_from_string; 5 fixed |
| [**cuda**](cuda/) | 1 Significant, 1 Minor | nccl communicators, CUDAPluggableAllocator shared_ptr; 1 fixed |
| [**distributed**](distributed/) | 7 SEVERE, 14 Significant, 18 Minor | groupRanks static race, SymmetricMemory missing mutex, PythonRpcHandler UAF, RRef dangling `this`; 1 fixed |
| [**inductor**](inductor/) | 8 SEVERE, 6 Significant, 6 Minor | aoti_kernel_cache_ zero synchronization, model_container partial locking, load_json_file static, runner registry |

## Broad audit only (lower quality, not deep-audited)

| Component | Issues | Details |
|-----------|--------|---------|
| [jit](jit/) | 8 medium | Mostly pure C++ graph transforms |
| [jit_mobile](jit_mobile/) | 4 medium | Mostly pure C++ |
| [fx](fx/) | 1 high, 2 medium | Lazy-init PyObject* |
| [export](export/) | 1 medium | upgrader_registry |
| [tensor](tensor/) | 1 medium | default_backend |

## No issues found

`api/` (C++ frontend), `cpu/`, `functionalization/`, `functorch/`, `instruction_counter/`, `lazy/` (already mutex-protected), `monitor/`, `mps/`, `mtia/`, `multiprocessing/`, `onnx/`, `stable/`.
