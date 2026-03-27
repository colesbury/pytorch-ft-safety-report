# Free-Threading Safety Audit: jit_passes__part1

**Overall Risk: LOW-MEDIUM**

Most files in this group are pure C++ graph transformations with no Python C API usage and no static/global mutable state. The two notable exceptions are `annotate_warns.cpp` (static atomic counter -- safe but monotonically growing) and `autocast.cpp` (static `std::atomic<bool>` for the autocast enabled flag, plus a static `RegisterOperators` in `batch_mm.cpp`).

---

## File Analysis

### jit/passes/add_if_then_else.cpp, .h
- **Risk: NONE**
- Pure graph transformation. No static/global state. No Python C API. Operates on a `shared_ptr<Graph>`.

### jit/passes/annotate_warns.cpp, .h
- **Risk: LOW**
- Contains `static std::atomic<int64_t> idx(0)` inside the `AnnotateWarns(Block*)` function (line 8). This is used to assign monotonically increasing `warn_id` values to `aten::warn` nodes.
- **Analysis:** The `std::atomic` ensures no data race on the counter itself. Under free-threading, concurrent calls to `AnnotateWarns` will produce unique IDs (no duplicates), but the IDs may be interleaved across graphs. This is functionally acceptable since the IDs just need to be unique, not sequential per graph.
- No Python C API interaction.

### jit/passes/autocast.cpp, .h
- **Risk: LOW**
- `static std::atomic<bool> autocast_enabled = true` (line 20): Global flag accessed via `setAutocastMode()` and `autocastEnabled()`. The `std::atomic` ensures safe concurrent reads/writes. The `exchange` in `setAutocastMode` is correct.
- The `Autocast()` pass reads `at::autocast::is_autocast_enabled()` and `at::autocast::get_autocast_dtype()` which are thread-local in eager mode. Under free-threading, concurrent JIT compilation using these thread-local values is fine because each thread reads its own autocast state.
- The pass itself (`handleBlock`) is a pure graph transformation operating on a single `shared_ptr<Graph>`. No Python C API.
- No concerns with the `AutocastContext` / `AutocastScope` structs -- they are stack-local.

### jit/passes/bailout_graph.cpp, .h
- **Risk: NONE**
- All state is local to the `BailOutGraphBuilderForNode` and `BailOutInserter` structs, which are created per-invocation. No static/global mutable state. No Python C API.

### jit/passes/batch_mm.cpp, .h
- **Risk: LOW**
- `static RegisterOperators mm_tree_reduction_reg(...)` (line 113): Static initialization of operator registration. This runs at dynamic library load time (before `main`), so it is not a free-threading concern -- it uses the standard operator registration mechanism.
- `static RegisterOperators mm_batch_side_reg(...)` (line 326): Same pattern as above.
- `static constexpr size_t min_fusion_size = 4` (line 83): Immutable constant, safe.
- The `TreeToken` struct and all pass logic are stack-local. No Python C API.

### jit/passes/canonicalize.cpp, .h
- **Risk: NONE**
- Pure graph transformations with all state on the stack. No static/global mutable state. No Python C API.

### jit/passes/canonicalize_graph_fuser_ops.cpp, .h
- **Risk: NONE**
- Pure graph transformations. No static/global mutable state. No Python C API.

### jit/passes/check_strict_fusion.cpp, .h
- **Risk: NONE**
- Pure graph inspection pass. All state is stack-local. No static/global mutable state. No Python C API.

---

## Summary of Issues

| File | Issue | Severity | Description |
|------|-------|----------|-------------|
| annotate_warns.cpp | Static `std::atomic<int64_t>` counter | Low | Already atomic; IDs are unique but may interleave across concurrent compilations. Acceptable. |
| autocast.cpp | Static `std::atomic<bool>` flag | Low | Already atomic; `exchange()` used correctly. No race. |
| batch_mm.cpp | Static `RegisterOperators` | Low | Runs at static init time. Uses standard registration. Safe. |

## Recommendations

1. **annotate_warns.cpp:** No action needed. The atomic counter is sufficient. If strictly sequential per-graph IDs are desired (unlikely needed), a per-graph counter could be used instead, but this is a cosmetic concern.
2. **autocast.cpp:** No action needed. The atomic boolean is thread-safe.
3. **batch_mm.cpp:** No action needed. Static operator registration at load time is safe.

No Python C API usage was found in any file in this group, so no `Py_BEGIN_CRITICAL_SECTION` concerns apply.
