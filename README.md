# PyTorch Free-Threading Safety Audit

Tracking thread-safety issues in `torch/csrc/` for Python 3.14t (free-threading / nogil).

See [CLAUDE.md](CLAUDE.md) for the audit methodology.

## Concurrency Model

- **Tier 1 (urgent):** `torch.compile()` runs on the main thread; other threads do data loading. The main risk is dict watcher callbacks firing on data loader threads.
- **Tier 2 (goal):** Multiple threads call compiled functions concurrently.

## Summary by Component (deep-audited)

| Component | Tier 1 | Tier 2 | Details |
|-----------|--------|--------|---------|
| [**dynamo**](dynamo/) | 8 SEVERE, 1 Significant, 2 Minor | 4 SEVERE, 12 Significant, 9 Minor | Dict watcher races, ExtraState, compiled autograd |
| [**profiler**](profiler/) | 2 SEVERE, 3 Significant, 1 Minor | — | profiler_kineto shared_ptr, GC callback UAF; 2 SEVERE already fixed |
| [**autograd**](autograd/) | 1 Significant, 1 Minor | 5 Significant, 2 Minor | cpp_function_types_map, lazy-init patterns |
| [**toplevel**](toplevel/) | 2 Significant | 2 Significant, 2 Minor | DataLoader worker_pids, InternedStringsTable |
| [**utils**](utils/) | 2 Significant, 1 Minor | 3 SEVERE, 1 Significant | device_lazy_init, python_dispatch maps |
| [**cuda**](cuda/) | 1 Minor | 1 Significant | CUDAPluggableAllocator shared_ptr |

## Broad audit only (lower quality, not deep-audited)

| Component | Issues | Details |
|-----------|--------|---------|
| [jit](jit/) | 8 medium | Mostly pure C++ graph transforms |
| [jit_mobile](jit_mobile/) | 4 medium | Mostly pure C++ |
| [fx](fx/) | 1 high, 2 medium | Lazy-init PyObject* |
| [export](export/) | 1 medium | upgrader_registry |
| [tensor](tensor/) | 1 medium | default_backend |

## No issues found

`api/` (C++ frontend), `cpu/`, `distributed/` (already mutex-protected), `functionalization/`, `functorch/`, `inductor/` (low Python coupling), `instruction_counter/`, `lazy/` (already mutex-protected), `monitor/`, `mps/`, `mtia/`, `multiprocessing/`, `onnx/`, `stable/`.
