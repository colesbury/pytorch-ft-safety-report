# Free-Threading Safety Audit: JIT IR Part 2

**Overall Risk: LOW**

## Files Reviewed

- `torch/csrc/jit/ir/node_hashing.cpp`
- `torch/csrc/jit/ir/node_hashing.h`
- `torch/csrc/jit/ir/scope.cpp`
- `torch/csrc/jit/ir/scope.h`
- `torch/csrc/jit/ir/subgraph_matcher.cpp`
- `torch/csrc/jit/ir/subgraph_matcher.h`
- `torch/csrc/jit/ir/type_hashing.cpp`
- `torch/csrc/jit/ir/type_hashing.h`

## Summary

These files implement JIT IR utilities: node hashing/equality for CSE, scope/callstack tracking, subgraph pattern matching, and type hashing. None of them interact with Python C API objects (no `PyObject*`, no GIL usage, no Python container access). They operate entirely on C++ JIT IR data structures.

There is no global or static mutable state in any of these files. All functions either operate on their arguments (pure functions) or use instance-local state (the `SubgraphMatcher` class, which is created locally in `findPatternMatches` and never shared across threads).

## Detailed Analysis

### node_hashing.cpp / node_hashing.h

`HashNode::operator()` and `EqualNode::operator()` are stateless functors that read from `Node*` arguments. All helper functions (`tensorEqual`, `typeListEqual`, `attributesEqual`, `ivaluesEqual`, `attributesEqualCSE`) are in an anonymous namespace and are pure functions operating on their arguments. No static mutable state. No thread-safety concerns.

### scope.cpp / scope.h

`Scope` and `InlinedCallStack` are intrusive-pointer-managed objects. All methods operate on instance state (`parent_`, `name_`, `callee_`, etc.) and do not modify any shared global state.

One static local: `Scope::isBlank()` contains `static const Symbol blank = Symbol::scope("")`. This is a `const` value initialized via C++11 magic statics (thread-safe initialization). Read-only after construction. **No issue.**

`InlinedCallStack::setCallee()` mutates instance state but this is a normal instance method -- callers are responsible for not sharing the object across threads without synchronization, which is the same contract as before.

### subgraph_matcher.cpp / subgraph_matcher.h

`SubgraphMatcher` is a class defined in an anonymous namespace, instantiated locally in `findPatternMatches`. Its mutable state (`nodes_map_`, `values_map_`, `anchor_`) is entirely local to the function call. The public API `findPatternMatches` takes `Graph&` references but does not modify them (the non-const `Graph&` is for returning non-const pointers in the `Match` results). No global or static mutable state. **No issue.**

### type_hashing.cpp / type_hashing.h

`HashType` and `EqualType` are stateless functors. The internal `hashType` function is a pure recursive function in an anonymous namespace. No static mutable state. **No issue.**

## Issues Found

None.

## Notes

These files form a clean, self-contained set of JIT IR utilities with no Python interop and no shared mutable state. Thread safety depends entirely on callers not sharing JIT IR graph structures (`Node*`, `Graph&`, `Scope`, `InlinedCallStack`) across threads without external synchronization, which is the existing contract for JIT IR objects.
