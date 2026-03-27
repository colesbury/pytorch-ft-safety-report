# Free-Threading Safety Audit: jit_testing

## Files Reviewed
- `torch/csrc/jit/testing/file_check.cpp`
- `torch/csrc/jit/testing/file_check.h`
- `torch/csrc/jit/testing/hooks_for_testing.cpp`
- `torch/csrc/jit/testing/hooks_for_testing.h`

## Overview

This group contains testing utilities: `FileCheck` (a pattern-matching tool for verifying IR output) and emit hooks for testing. These are testing-only infrastructure, not production code paths.

## Issues

### Issue 1: `emit_module_callback` and `emit_function_callback` are unsynchronized globals

**Location:** `hooks_for_testing.cpp`, lines 7-8, 14-15

```cpp
static ModuleHook emit_module_callback;
static FunctionHook emit_function_callback;
```

These are `std::function<void(...)>` objects that are:
- Written by `setEmitHooks()` (line 21)
- Read by `didFinishEmitModule()` (line 9) and `didFinishEmitFunction()` (line 15)
- Read by `getEmitHooks()` (line 26)

`std::function` is not thread-safe for concurrent read/write. Without the GIL, concurrent calls to `setEmitHooks` and `didFinishEmitModule`/`didFinishEmitFunction` could cause data races (reading the function object while it is being replaced).

**Risk:** Medium -- if these hooks are set from Python (which they likely are, since they are testing hooks) and emit callbacks fire from compilation threads, there is a genuine race. However, in practice, hooks are typically set once before any compilation.

**Recommended Fix:** Protect with a mutex, or use `std::atomic<std::shared_ptr<...>>` to allow lock-free reads.

### Issue 2: `FileCheck` is purely local state, no thread-safety concerns

**Location:** `file_check.cpp`, `file_check.h`

`FileCheck` instances hold all state in the `FileCheckImpl` object owned by `unique_ptr`. There is no shared global state. Each instance is used independently.

The `static const std::vector<...> check_pairs` on line 245 is a function-local static const, initialized thread-safely.

**Risk:** None.

### Issue 3: `FileCheck::has_run` member in header

**Location:** `file_check.h`, line 75

```cpp
bool has_run = false;
```

This is a per-instance field (not static), so there is no cross-thread concern as long as a single `FileCheck` instance is not shared between threads (which would be an unusual usage pattern).

**Risk:** None.

## Summary

The only issue is the unsynchronized global `std::function` callbacks in `hooks_for_testing.cpp` (Issue 1). These are testing-only hooks, but they are technically racy if `setEmitHooks` is called concurrently with `didFinishEmitModule`/`didFinishEmitFunction`. The `FileCheck` utility has no thread-safety concerns as it operates on purely local state. No Python C API is used in any of these files.
