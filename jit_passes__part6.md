# Free-Threading Safety Audit: jit_passes__part6

**Overall Risk: LOW**

## Files Audited

- `jit/passes/guard_elimination.cpp`
- `jit/passes/guard_elimination.h`
- `jit/passes/hoist_conv_packed_params.cpp`
- `jit/passes/hoist_conv_packed_params.h`
- `jit/passes/inline_autodiff_subgraphs.cpp`
- `jit/passes/inline_autodiff_subgraphs.h`
- `jit/passes/inline_fork_wait.cpp`
- `jit/passes/inline_fork_wait.h`
- `jit/passes/inline_forked_closures.cpp`
- `jit/passes/inline_forked_closures.h`
- `jit/passes/inliner.cpp`
- `jit/passes/inliner.h`
- `jit/passes/inplace_check.cpp`
- `jit/passes/inplace_check.h`
- `jit/passes/insert_guards.cpp`

## Detailed Analysis

### guard_elimination.cpp/.h

The `GuardElimination` struct takes a `shared_ptr<Graph>` and creates a local `AliasDb`. All mutable state (`graph_`, `aliasDb_`) is instance-local. The `static std::unordered_set<Symbol> simple_ops_` declaration at line 454 is never defined or used -- it appears to be dead code with no initialization.

The `removableGuard` method has `const static auto no_exceptions = std::unordered_set<size_t>{}` (line 280), which is a const static local, safely initialized once per C++11.

**Verdict: SAFE** -- All state is local to the `GuardElimination` instance.

### hoist_conv_packed_params.cpp/.h

`HoistConvPackedParams` operates on a `script::Module&` passed in. All state (blocks_to_visit stack, nameUniqueCounter, attr_name_base) is local to the function. The `hoistConvPackedParams` helper modifies the module and graph in place but uses only local state.

**Verdict: SAFE** -- No shared mutable state, no Python C API.

### inline_autodiff_subgraphs.cpp/.h

Pure graph traversal pass. `InlineAutodiffSubgraphs` recursively scans blocks and inlines small differentiable subgraphs. `canRunWithAutograd` is a stateless predicate. All state is function-local.

**Verdict: SAFE**

### inline_fork_wait.cpp/.h

`InlineForkWait` uses a local `std::unordered_map<Value*, Value*> future_remap` passed by reference through the recursive calls. All state is call-local.

**Verdict: SAFE**

### inline_forked_closures.cpp/.h

`inlineForkedClosures` recursively traverses blocks and processes `prim::forkClosure` and `prim::awaitableClosure` nodes. All state is local (the recursive traversal, graph cloning).

**Verdict: SAFE**

### inliner.cpp/.h

`Inline(Graph&)` and `tryToGraphFunction(Node*)` are stateless traversals that inline function/method calls. `inlineCalls` is a recursive block traversal with local state only. The function accesses `GraphFunction::optimized_graph()` and `get_executor()` which have their own thread-safety considerations, but the `inliner.cpp` itself introduces no new shared state.

**Verdict: SAFE**

### inplace_check.cpp/.h

`CheckInplace` is a trivial traversal that checks for inplace `PythonOp` nodes. No mutable state.

**Verdict: SAFE**

### insert_guards.cpp

The `GuardInserter` struct holds a `shared_ptr<Graph>` member and traverses blocks to insert guard nodes. All state is instance-local. `InsertGuards` creates a local instance and calls `run()`.

**Verdict: SAFE**

## Issues Found

No thread-safety issues found in this group.

## Summary

All files in this group are pure C++ graph transformation passes that operate on graph/module objects passed in as arguments. They maintain only local/instance state, make no Python C API calls, and have no static or global mutable state. This group is safe for free-threading.
