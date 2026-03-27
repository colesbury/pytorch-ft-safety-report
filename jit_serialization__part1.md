# Free-Threading Safety Audit: jit_serialization__part1

## Files Audited
- `torch/csrc/jit/serialization/callstack_debug_info_serialization.cpp`
- `torch/csrc/jit/serialization/callstack_debug_info_serialization.h`
- `torch/csrc/jit/serialization/export.cpp`
- `torch/csrc/jit/serialization/export.h`
- `torch/csrc/jit/serialization/export_bytecode.cpp`
- `torch/csrc/jit/serialization/export_bytecode.h`
- `torch/csrc/jit/serialization/export_module.cpp`
- `torch/csrc/jit/serialization/flatbuffer_serializer.cpp`
- `torch/csrc/jit/serialization/flatbuffer_serializer.h`
- `torch/csrc/jit/serialization/flatbuffer_serializer_jit.cpp`
- `torch/csrc/jit/serialization/flatbuffer_serializer_jit.h`
- `torch/csrc/jit/serialization/import.cpp`
- `torch/csrc/jit/serialization/import.h`
- `torch/csrc/jit/serialization/import_export_constants.h`
- `torch/csrc/jit/serialization/import_export_functions.h`

## Summary

This group covers JIT serialization infrastructure: export (ONNX, TorchScript), bytecode compilation, flatbuffer serialization, and import/load. These files are primarily C++ with no direct Python C API usage. The main thread-safety concerns are static mutable global state used for configuration and hook registration.

## Issues Found

### Issue 1: Static mutable `ExportModuleExtraFilesHook` without synchronization
**File:** `torch/csrc/jit/serialization/export_module.cpp` (line 77-79)
**Severity:** Medium
**Description:** `GetExtraFilesHook()` returns a reference to a `static ExportModuleExtraFilesHook func` (a `std::function`). `SetExportModuleExtraFilesHook` writes to this via move-assignment, and it is read during `serialize()` calls. Without the GIL serializing access, concurrent set/read is a data race.
```cpp
ExportModuleExtraFilesHook& GetExtraFilesHook() {
  static ExportModuleExtraFilesHook func = nullptr;
  return func;
}
```
**Recommendation:** Protect with a mutex, or make the hook storage use `std::atomic` with a shared_ptr wrapper.

### Issue 2: Static mutable configuration flags in `BytecodeEmitMode`
**File:** `torch/csrc/jit/serialization/export_module.cpp` (lines 959, 969, 978)
**Severity:** Low
**Description:** Three `static thread_local bool` variables (`emitBytecodeDefaultInputs`, `emitDefautlArgsWithOutArgs`, `emitDefaultEmitPromotedOps`) are used to control bytecode emission. These are `thread_local`, so they are already thread-safe -- each thread has its own copy. The `BytecodeEmitModeGuard` RAII guard in `export.h` also works correctly with thread-local storage.
**Status:** Safe. No action needed.

### Issue 3: `mobileInterfaceCallExport` uses `std::atomic<bool>` correctly
**File:** `torch/csrc/jit/serialization/export_module.cpp` (line 431)
**Severity:** None
**Description:** The `mobileInterfaceCallExport()` flag uses `std::atomic<bool>` with `memory_order_relaxed`, which is safe for a simple boolean flag.
**Status:** Safe. No action needed.

### Issue 4: Static local string constants in bytecode generation
**File:** `torch/csrc/jit/serialization/export_module.cpp` (lines 194-195)
**Severity:** None
**Description:** `static const std::string torch_prefix` and `class_prefix` are immutable after initialization. Safe for concurrent reads.
**Status:** Safe.

### Issue 5: Static local string constants in flatbuffer serializer
**File:** `torch/csrc/jit/serialization/flatbuffer_serializer.cpp` (lines 281-282)
**Severity:** None
**Description:** Same pattern as above -- `static const std::string` values. Immutable after initialization.
**Status:** Safe.

### Issue 6: Static local `patterns_and_replacements` in import.cpp
**File:** `torch/csrc/jit/serialization/import.cpp` (line 230)
**Severity:** None
**Description:** `static const std::vector<...> patterns_and_replacements` is const-qualified and immutable. Safe.
**Status:** Safe.

## Concurrency Notes

- The callstack serialization/deserialization classes (`InlinedCallStackSerializer`, `InlinedCallStackDeserializer`, `CallStackDebugInfoPickler`, `CallStackDebugInfoUnpickler`) use instance-level caches (member variables). They are not thread-safe for concurrent use of a single instance, but this is by design -- each serialization/deserialization operation should use its own instance.
- The `ScriptModuleSerializer` class in `export.h` stores per-serialization state in member variables. Thread safety depends on not sharing a single instance across threads, which is the expected usage pattern.
- None of these files contain Python C API calls (`PyObject*`, `PyGILState_Ensure`, etc.). The serialization layer operates at the C++ IValue/Module level.
