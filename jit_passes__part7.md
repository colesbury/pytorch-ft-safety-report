# Free-Threading Safety Audit: jit_passes__part7

**Overall Risk: LOW**

## Files Audited

- `jit/passes/insert_guards.h`
- `jit/passes/integer_value_refinement.cpp`
- `jit/passes/integer_value_refinement.h`
- `jit/passes/lift_closures.cpp`
- `jit/passes/lift_closures.h`
- `jit/passes/liveness.cpp`
- `jit/passes/liveness.h`
- `jit/passes/loop_unrolling.cpp`
- `jit/passes/loop_unrolling.h`
- `jit/passes/lower_grad_of.cpp`
- `jit/passes/lower_grad_of.h`
- `jit/passes/lower_graph.cpp`
- `jit/passes/lower_graph.h`
- `jit/passes/lower_tuples.cpp`
- `jit/passes/lower_tuples.h`

## Detailed Analysis

### insert_guards.h

Header-only declarations. No mutable state.

**Verdict: SAFE**

### integer_value_refinement.cpp/.h

The `IntegerValueRefiner` struct operates on a `shared_ptr<Graph>` with all state instance-local: `active_refinements_` (stack of refinement maps), `info_` (map from Value* to refinement info), `throwing_blocks_`, `changed_`. The entry point `RefineIntegerValues` creates a local instance and calls `run()`.

**Verdict: SAFE** -- All state is instance-local.

### lift_closures.cpp/.h

`liftClosures` recursively traverses blocks, converting closure blocks into subgraphs. All state is local to the recursive call (captures map, graph nodes). No shared mutable state.

**Verdict: SAFE**

### liveness.cpp/.h

The `LivenessAnalyzer` struct holds instance-local state: `graph_`, `changed_`, `liveness_sets_`, `ids_to_values_`. The entry point `BuildLivenessSets` creates a local instance. The fixed-point analysis loop modifies only instance state.

**Verdict: SAFE** -- All state is instance-local.

### loop_unrolling.cpp/.h

Contains several file-scope constants (`kUnrollFactor`, `kMaxBodySize`, `kMaxBodyRepeats`) that are `static constexpr` -- immutable.

All functions (`UnrollLoops`, `UnrollConstantLoops`, `PeelLoop`, `PeelProfilingLoops`) operate on passed-in graph objects with local state. The `LoopsPeeler` class holds instance state (`callback_`, `in_loop_`, `loops_to_peel_`, `num_iterations_`).

The `PeelLoop` function has `static const size_t LOOP_DEPS_WITH_COND_OFFSET = 2` (line 349) which is a const -- safe.

**Verdict: SAFE** -- All mutable state is local/instance-scoped.

### lower_grad_of.cpp/.h

`LowerGradOf` operates on a `Graph&` directly, iterating over nodes and replacing `prim::GradOf` nodes with conditionals. All state is local.

**Verdict: SAFE**

### lower_graph.cpp/.h

`LowerGraph` and the internal `lower_graph` operate on a `Graph&` and `ModulePtr`. All state (`extra_ivalues`, `slot_to_offset`, `to_scan`, `to_clean`) is local to the function. The recursive call for `prim::fork` subgraphs is self-contained.

**Verdict: SAFE**

### lower_tuples.cpp/.h

The file has a namespace-scope `std::unordered_set<Symbol> supported_ops` (line 18), but this set is declared as a non-local variable in an anonymous namespace and initialized with a fixed initializer list. Since it is only read (via `supported_ops.count(...)`) and never modified after initialization, it is effectively const.

All lowering functions (`LowerAllTuples`, `LowerSimpleTuples`, `removeTupleNodes`, etc.) operate on passed-in blocks/graphs with local state.

**Verdict: SAFE** -- The `supported_ops` set is effectively const after static initialization.

## Issues Found

No thread-safety issues found in this group.

## Summary

All files in this group are pure C++ graph transformation and analysis passes. They operate on graph objects passed in as arguments, with all mutable state scoped to function locals or struct instances. There are no Python C API calls, no mutable static/global state, and no lazy initialization patterns. This group is safe for free-threading.
