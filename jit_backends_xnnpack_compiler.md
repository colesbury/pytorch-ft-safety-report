# Free-Threading Safety Audit: jit/backends/xnnpack/compiler

**Overall Risk: Low**

## Files Reviewed
- `torch/csrc/jit/backends/xnnpack/compiler/xnn_compiler.cpp`
- `torch/csrc/jit/backends/xnnpack/compiler/xnn_compiler.h`

## Findings

### 1. XNNCompiler::compileModel() -- Static method, all local state (Safe)
**File:** `xnn_compiler.cpp`, lines 16-116
```cpp
void XNNCompiler::compileModel(
    const void* buffer_pointer,
    size_t num_bytes,
    XNNExecutor* executor) {
  ...
}
```
This is a static method that:
- Reads from a flatbuffer (`buffer_pointer`)
- Creates local XNNPACK subgraph and runtime objects
- Populates the output `executor` parameter

All intermediate state (`remapped_ids`, `subgraph_ptr`, `runtime_ptr`, local vectors) is stack-local. The function writes to the output `executor` parameter, but the caller is responsible for ensuring exclusive access to that executor during compilation.

The `xnn_initialize()` call at line 25 is an XNNPACK library call. Per XNNPACK documentation, it is safe to call multiple times. However, concurrent `xnn_initialize` + `xnn_create_subgraph` + `xnn_create_runtime_v2` calls depend on XNNPACK's own thread safety guarantees.

**Severity:** None for PyTorch-level thread safety. The function introduces no shared mutable C++ state.

### 2. XNNCompiler class -- No member variables (Safe)
**File:** `xnn_compiler.h`
```cpp
class XNNCompiler {
 public:
  static void compileModel(
      const void* buffer_pointer,
      size_t num_bytes,
      XNNExecutor* executor);
};
```
`XNNCompiler` is a stateless utility class with only a static method. No instance or static mutable state.

**Severity:** None

## Summary

This group is safe from a PyTorch free-threading perspective. `XNNCompiler` is a stateless utility class with a single static method that uses only local variables. It introduces no shared mutable C++ state. Thread safety of the underlying XNNPACK library calls is outside the scope of this audit.
