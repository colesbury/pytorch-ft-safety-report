# Free-Threading Safety Audit: jit_codegen_onednn__part2

## Files Reviewed
- `torch/csrc/jit/codegen/onednn/kernel.cpp`
- `torch/csrc/jit/codegen/onednn/kernel.h`
- `torch/csrc/jit/codegen/onednn/layout_propagation.cpp`
- `torch/csrc/jit/codegen/onednn/layout_propagation.h`
- `torch/csrc/jit/codegen/onednn/operator.h`
- `torch/csrc/jit/codegen/onednn/prepare_binary.cpp`
- `torch/csrc/jit/codegen/onednn/prepare_binary.h`
- `torch/csrc/jit/codegen/onednn/register_interface.cpp`

## Overview

This group contains the oneDNN Graph kernel execution, layout propagation, binary op preparation, and the profiling node registration interface. The `LlgaKernel` class is the main runtime component that compiles and runs oneDNN Graph partitions.

## Issues

### Issue 1: `LlgaKernel::genDebugName` has unsynchronized static counter

**Location:** `kernel.h`, lines 55-58

```cpp
static std::string genDebugName() {
  static size_t debugId = 0;
  return "LlgaPartition_" + std::to_string(debugId++);
}
```

The `debugId++` is a non-atomic increment of a static local variable. Under free-threading, concurrent `LlgaKernel` construction could race on this counter.

**Risk:** Low -- this is only for debug naming; duplicate or skipped IDs cause no functional harm.

**Recommended Fix:** Use `static std::atomic<size_t> debugId{0}`.

### Issue 2: `LlgaKernel::run` uses `c10::call_once` for initialization

**Location:** `kernel.cpp`, lines 256-267

```cpp
c10::call_once(
    initialized_flag,
    [&](const TensorArgs& inputs) {
      inputSpecs_ = initializeInputSpecs(inputs);
      outputSpecs_ = initializeOutputSpecs();
      compilation_ = compile(partition_);
      is_initialized_ = true;
    },
    inputs);
```

This correctly uses `c10::call_once` / `c10::once_flag` for lazy initialization. The member fields `inputSpecs_`, `outputSpecs_`, and `compilation_` are written inside the once-block and read afterward. The `is_initialized_` flag is set inside the once block. This pattern is safe: `call_once` provides the necessary memory ordering.

However, the fields `inputSpecs_` and `outputSpecs_` are also read in `prepareRunArgs()` (line 121) after the `call_once` returns. Since `call_once` establishes a happens-before relationship, this is correct.

**Risk:** None.

### Issue 3: `register_interface_` static registration

**Location:** `register_interface.cpp`, lines 38-46

```cpp
static RegisterInterface register_interface_;
```

The constructor calls `RegisterProfilingNode(canFuseNode)`. This runs at static initialization time before user threads.

**Risk:** None.

### Issue 4: Layout propagation, binary preparation, and operator mapping are all pure graph transformations

**Location:** `layout_propagation.cpp`, `prepare_binary.cpp`, `operator.h`

These functions operate on a `shared_ptr<Graph>` or individual `Node*` objects. They do not access global mutable state. Each invocation works on its own graph.

**Risk:** None.

### Issue 5: Engine and Stream singletons in LlgaTensorImpl.cpp (from part1)

These are Meyers' singletons used in `prepareRunArgs`:

```cpp
static dnnl::engine cpu_engine = ...;
static dnnl::stream cpu_stream{Engine::getEngine()};
```

The engine and stream are created once (thread-safe init) and used concurrently. Whether the oneDNN stream/engine objects are themselves thread-safe for concurrent use depends on the oneDNN library. The code comment notes this stream is shared, which may not be safe if oneDNN requires per-thread streams for concurrent execution.

**Risk:** Medium -- depends on oneDNN's thread-safety guarantees for shared `dnnl::stream` objects. If oneDNN requires per-thread streams, this is unsafe under concurrent execution.

## Summary

The main issues are:

1. **`genDebugName` non-atomic counter** -- trivially fixable with `std::atomic`, low risk.
2. **Shared `dnnl::stream` singleton** -- needs verification against oneDNN's threading model. If oneDNN requires per-thread streams, the shared singleton could be unsafe under concurrent kernel execution.

The `LlgaKernel::run` method properly uses `c10::call_once` for lazy initialization. No Python C API is used anywhere in these files.
