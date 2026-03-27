# Free-Threading Safety Audit: jit_mobile__part1

## Files Audited
- `jit/mobile/code.h`
- `jit/mobile/debug_info.cpp`
- `jit/mobile/debug_info.h`
- `jit/mobile/file_format.h`
- `jit/mobile/flatbuffer_loader.cpp`
- `jit/mobile/flatbuffer_loader.h`
- `jit/mobile/frame.h`
- `jit/mobile/function.cpp`
- `jit/mobile/function.h`
- `jit/mobile/import.cpp`
- `jit/mobile/import.h`
- `jit/mobile/import_data.cpp`
- `jit/mobile/import_data.h`
- `jit/mobile/import_export_common.h`
- `jit/mobile/interpreter.cpp`

## Summary

These files implement the core mobile/lite interpreter for PyTorch, including model deserialization (ZIP pickle and flatbuffer formats), bytecode execution, and the function/module abstractions. The code is predominantly C++-only (no Python C API usage) and operates on deserialized model state. The main free-threading concerns involve static mutable state and lazy initialization patterns in the C++ layer.

## Findings

### 1. Static mutable upgrader function holder (function.cpp)
- **File**: `jit/mobile/function.cpp`, lines 248-271
- **Severity**: Medium
- **Description**: `Function::registerFunc` uses a `static std::unordered_map<c10::QualifiedName, Function> upgrader_function_holder` that is accessed and mutated without synchronization. Multiple threads loading models concurrently would race on `find` / `emplace` operations on this map.
- **Current GIL Dependence**: This code is typically called during model loading, which may have been serialized by the GIL.
- **Recommendation**: Protect with a `std::mutex` or convert to a thread-safe container. Since this is a lazy-populate-once pattern, a `std::call_once` approach per entry or a global lock during registration would work.

### 2. Static `torchPrefix` and `jitPrefix` in import.cpp (safe)
- **File**: `jit/mobile/import.cpp`, lines 98-99
- **Severity**: None
- **Description**: `static const c10::QualifiedName torchPrefix` and `jitPrefix` are `const` statics initialized with constant string arguments. These are safe for concurrent reads. The initialization is thread-safe (C++11 static local guarantee).

### 3. Static `torchPrefix` in import_data.cpp (safe)
- **File**: `jit/mobile/import_data.cpp`, line 81
- **Severity**: None
- **Description**: Similar to import.cpp, this is a `static const c10::QualifiedName` inside a lambda/function scope. Thread-safe initialization.

### 4. Static constexpr strings in flatbuffer_loader.cpp (safe)
- **File**: `jit/mobile/flatbuffer_loader.cpp`, lines 73-76
- **Severity**: None
- **Description**: `kCustomClassPrefix`, `kTorchPrefix`, `kJitPrefix` are `static constexpr std::string_view` values. Immutable and safe.

### 5. `std::rand()` usage in import.cpp
- **File**: `jit/mobile/import.cpp`, lines 504, 683
- **Severity**: Low
- **Description**: `std::rand()` is used to generate `instance_key` values for observer callbacks. `std::rand()` is not thread-safe as it modifies global PRNG state. Under free-threading, concurrent calls could produce garbled results or data races on internal PRNG state.
- **Recommendation**: Replace with a thread-safe RNG such as `std::mt19937` with thread-local state, or use `c10::detail::getNonDeterministicRandom()`.

### 6. `observerConfig()` global singleton (observer.cpp)
- **File**: `jit/mobile/observer.cpp`, lines 5-8
- **Severity**: Low
- **Description**: `observerConfig()` returns a reference to a `static MobileObserverConfig instance`. The static local initialization is thread-safe (C++11 guarantee). However, the `MobileObserverConfig` object itself stores a `std::unique_ptr<MobileModuleObserver>` that is set via `setModuleObserver()` and read via `getModuleObserver()`. If one thread configures the observer while another reads it, there is a data race.
- **Recommendation**: This is a configuration-time vs. runtime issue. If the observer is always set before any model loading occurs (typical usage), this is fine in practice. If not, protect with a mutex or use `std::atomic<MobileModuleObserver*>`.

### 7. FlatbufferLoader is file-local and stack-allocated (safe)
- **File**: `jit/mobile/flatbuffer_loader.cpp`
- **Severity**: None
- **Description**: `FlatbufferLoader` is defined in an anonymous namespace and instantiated as a local variable inside `parse_and_initialize_mobile_module`. Each call gets its own instance. No shared mutable state across threads.

### 8. Thread-local `exception_debug_handles_` (interpreter.cpp)
- **File**: `jit/mobile/interpreter.cpp`, line 25
- **Severity**: None
- **Description**: `exception_debug_handles_` is declared `thread_local`, so each thread has its own copy. This is already free-threading safe.

## No Python C API Usage

None of the files in this group use the Python C API (no `PyObject*`, no `Py_BEGIN_CRITICAL_SECTION`, no `PyGILState_Ensure`, etc.). All code operates at the C++/ATen layer. Free-threading concerns are limited to C++ static/global mutable state.

## Overall Risk Assessment: LOW

The primary risk is the unprotected static map in `Function::registerFunc`. The `std::rand()` usage is a minor correctness issue. The observer config is a design pattern issue that is unlikely to cause problems in practice. The vast majority of code in this group uses local variables and per-instance state, making it inherently thread-safe.
