# Free-Threading Safety Audit: jit_mobile_train

## Files Audited
- `jit/mobile/train/export_data.cpp`
- `jit/mobile/train/export_data.h`
- `jit/mobile/train/random.cpp`
- `jit/mobile/train/random.h`
- `jit/mobile/train/sequential.cpp`
- `jit/mobile/train/sequential.h`

## Summary

This group implements on-device training support for mobile, including parameter serialization/deserialization (export_data), a random sampler, and a sequential sampler for data loading. The code is pure C++ with no Python C API usage. The samplers maintain per-instance state and the serialization code uses local variables.

## Findings

### 1. Global function pointer `_save_mobile_module_to` (export_data.cpp)
- **File**: `jit/mobile/train/export_data.cpp`, lines 114-116; `jit/mobile/train/export_data.h`, lines 47-48
- **Severity**: Low
- **Description**: `_save_mobile_module_to` is declared as a global function pointer:
  ```cpp
  void (*_save_mobile_module_to)(...) = nullptr;
  ```
  This pointer is set externally (likely by flatbuffer serializer code) and read by `_save_parameters()` via `save_mobile_module_to_func()`. If one thread sets this pointer while another reads it, there is a data race. However, this follows the typical "configure once at startup, use at runtime" pattern.
- **Recommendation**: If the pointer is always set during static initialization or module import before any concurrent use, this is safe in practice. Otherwise, use `std::atomic<void(*)(...)*>` or similar.

### 2. `RandomSampler` instance state (random.cpp)
- **File**: `jit/mobile/train/random.cpp`
- **Severity**: None
- **Description**: `RandomSampler` maintains per-instance state (`indices_`, `index_`). It is not designed to be shared across threads. Thread safety is the caller's responsibility, which is standard for data samplers.

### 3. `SequentialSampler` instance state (sequential.cpp)
- **File**: `jit/mobile/train/sequential.cpp`
- **Severity**: None
- **Description**: Like `RandomSampler`, `SequentialSampler` maintains per-instance state (`size_`, `index_`). Not designed for cross-thread sharing. Thread safety is the caller's responsibility.

### 4. `IValuePickler` in export_data.cpp (safe)
- **File**: `jit/mobile/train/export_data.cpp`, lines 26-69
- **Severity**: None
- **Description**: `IValuePickler` is a local class used within anonymous namespace. It is instantiated as a stack-local variable in `_save_parameters()`. Each call gets its own instance. No shared state.

### 5. `tensor_map_to_dict` and `tensor_dict_to_mobile` (safe)
- **File**: `jit/mobile/train/export_data.cpp`, lines 76-110
- **Severity**: None
- **Description**: Pure utility functions that create new objects from inputs. No shared mutable state. Thread-safe.

### 6. `_save_parameters` functions (safe)
- **File**: `jit/mobile/train/export_data.cpp`, lines 118-147
- **Severity**: None
- **Description**: These functions create local `IValuePickler` instances or call into `save_mobile_module_to_func`. They use local state (output streams). Thread-safe as long as the output stream/file is not shared.

## No Python C API Usage

None of the files in this group use the Python C API.

## Overall Risk Assessment: LOW

The training support code is straightforward with minimal shared state. The only concern is the global function pointer `_save_mobile_module_to`, which follows a safe usage pattern (set once, read many). The samplers use per-instance state and are not designed for concurrent access, which is the standard design for such classes.
