# Free-Threading Safety Audit: jit_serialization__part3

## Files Audited
- `torch/csrc/jit/serialization/python_print.cpp`
- `torch/csrc/jit/serialization/python_print.h`
- `torch/csrc/jit/serialization/source_range_serialization.cpp`
- `torch/csrc/jit/serialization/source_range_serialization.h`
- `torch/csrc/jit/serialization/source_range_serialization_impl.h`
- `torch/csrc/jit/serialization/storage_context.h`
- `torch/csrc/jit/serialization/type_name_uniquer.cpp`
- `torch/csrc/jit/serialization/type_name_uniquer.h`
- `torch/csrc/jit/serialization/unpickler.cpp`
- `torch/csrc/jit/serialization/unpickler.h`

## Summary

This group covers Python code generation from IR graphs (python_print), source range serialization/deserialization, storage context management, type name uniquing, and the unpickler. The main thread-safety concern is a mutable `thread_local` flag controlling serialization format.

## Issues Found

### Issue 1: `thread_local` flag `should_use_format_with_string_table_`
**File:** `torch/csrc/jit/serialization/source_range_serialization.cpp` (line 16)
**Severity:** Low
**Description:** `static thread_local bool should_use_format_with_string_table_ = true` is thread-local, so it is inherently thread-safe -- each thread has its own copy. The public API `setShouldUseFormatWithStringTable()` sets the calling thread's copy. This is safe under free-threading, but callers should be aware that the setting only applies to the calling thread.
**Status:** Safe. Thread-local storage is the correct approach here.

### Issue 2: `ConcreteSourceRangeUnpickler` uses mutex correctly
**File:** `torch/csrc/jit/serialization/source_range_serialization.cpp` (line 197), `source_range_serialization_impl.h` (line 23)
**Severity:** None
**Description:** `ConcreteSourceRangeUnpickler::unpickle()` uses a `std::mutex` to protect lazy initialization of `unpickled_records` and `deserializer`. This is already correctly synchronized. Subsequent reads of `unpickled_records` in `findSourceRangeThatGenerated` happen after `unpickle()` (which acquires the lock), so the data is safely published.
**Status:** Safe. Already properly synchronized.

### Issue 3: Static `const` data in python_print.cpp
**File:** `torch/csrc/jit/serialization/python_print.cpp` (lines 43, 981)
**Severity:** None
**Description:** `const static std::unordered_set<std::string> reserved_names` and `const static std::unordered_map<Symbol, std::string> override_symbols` are immutable after initialization. Thread-safe via magic statics for initialization, and safe for concurrent reads thereafter.
**Status:** Safe.

### Issue 4: Static member functions in python_print.cpp
**File:** `torch/csrc/jit/serialization/python_print.cpp` (lines 379, 927)
**Severity:** None
**Description:** `makeValidIdentifier` and `containsNonASCIIString` are pure static functions with no mutable state. Safe.
**Status:** Safe.

### Issue 5: Static `defaultTypeParser` in unpickler.h
**File:** `torch/csrc/jit/serialization/unpickler.h` (line 110)
**Severity:** None
**Description:** This is a static inline function that delegates to `c10::parseType`. No mutable state.
**Status:** Safe.

## Concurrency Notes

- `python_print.cpp` uses `PythonPrint` which accumulates output into member-level `OrderedDict` and streams. Each `PythonPrint` instance is used during a single serialization operation and is not shared across threads.
- `SourceRangeSerializer` and `SourceRangeDeserializer` use instance-level caches. Thread-safe when not shared across threads.
- `TypeNameUniquer` uses instance-level `std::unordered_map` for uniquing. Thread-safe when not shared across threads.
- `Unpickler` uses instance-level state (`stack_`, `memo_table_`, etc.). Thread-safe when not shared across threads, which is the expected usage.
- `storage_context.h` defines `SerializationStorageContext` and `DeserializationStorageContext` which use instance-level maps. Same instance-per-operation pattern.
- No Python C API calls in any of these files.
