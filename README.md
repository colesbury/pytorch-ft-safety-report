# PyTorch Free-Threading Safety Audit

Tracking thread-safety issues in `torch/csrc/` for Python 3.14t (free-threading / nogil).

See [CLAUDE.md](CLAUDE.md) for the audit methodology.

## Concurrency Model

- **Tier 1 (urgent):** `torch.compile()` runs on the main thread; other threads do data loading. The main risk is dict watcher callbacks firing on data loader threads.
- **Tier 2 (goal):** Multiple threads call compiled functions concurrently.

## Summary by Component

| Component | Tier 1 Issues | Tier 2 Issues | Status |
|-----------|--------------|---------------|--------|
| [**dynamo**](dynamo/) | 11 | 25 | Active |
| autograd | ? | 6 high, 17 medium | Broad audit only |
| torch/csrc (top-level) | ? | 3 high, 18 medium | Broad audit only |
| utils | ? | 4 high, 8 medium | Broad audit only |
| cuda | ? | 2 high, 4 medium | Broad audit only |
| profiler | ? | 0 high, 6 medium | Broad audit only |

## Components with No Issues

The following had no free-threading issues: `api/` (C++ frontend), `cpu/`, `functionalization/`, `functorch/`, `instruction_counter/`, `monitor/`, `mps/`, `mtia/`, `multiprocessing/`, `onnx/`, `stable/`.

## Reports

- [dynamo/](dynamo/) — deep audit with tier classification (v2)
- [CLAUDE.md](CLAUDE.md) — audit methodology and what to look for
- Individual broad-audit reports (`autograd__part1.md`, etc.) — initial pass, lower quality
