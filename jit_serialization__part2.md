# Free-Threading Safety Audit: jit_serialization__part2

## Files Audited
- `torch/csrc/jit/serialization/import_export_helpers.cpp`
- `torch/csrc/jit/serialization/import_export_helpers.h`
- `torch/csrc/jit/serialization/import_read.cpp`
- `torch/csrc/jit/serialization/import_read.h`
- `torch/csrc/jit/serialization/import_source.cpp`
- `torch/csrc/jit/serialization/import_source.h`
- `torch/csrc/jit/serialization/mobile_bytecode_generated.h`
- `torch/csrc/jit/serialization/onnx.cpp`
- `torch/csrc/jit/serialization/onnx.h`
- `torch/csrc/jit/serialization/pickle.cpp`
- `torch/csrc/jit/serialization/pickle.h`
- `torch/csrc/jit/serialization/pickler.cpp`
- `torch/csrc/jit/serialization/pickler.h`
- `torch/csrc/jit/serialization/pickler_helper.cpp`
- `torch/csrc/jit/serialization/pickler_helper.h`

## Summary

This group covers import helpers, source import (for reconstructing types from serialized Python source), ONNX pretty-printing, pickle/unpickle infrastructure, and flatbuffer-generated headers. Most of these files are stateless utility functions or use instance-level state. A few have lazy-initialized static data structures that merit review.

## Issues Found

### Issue 1: Static mutable `replacements` map in import_source.cpp
**File:** `torch/csrc/jit/serialization/import_source.cpp` (line 319)
**Severity:** None
**Description:** A `static std::unordered_map<std::string, AttrTypeReplacementDescr> replacements{...}` is defined inside a function. This is a const-like initialization -- the map is populated at initialization time via aggregate initialization and never modified afterwards. The C++11 "magic statics" guarantee ensures the initialization is thread-safe. Subsequent concurrent reads are safe.
**Status:** Safe.

### Issue 2: Static `std::regex` in import_source.cpp
**File:** `torch/csrc/jit/serialization/import_source.cpp` (line 370)
**Severity:** None
**Description:** `static std::regex mangle_re(...)` is initialized once via magic statics. `std::regex_replace` with a const regex is safe for concurrent use in libstdc++ and libc++ (the regex itself is not modified).
**Status:** Safe.

### Issue 3: Static `QualifiedName` in import_source.cpp
**File:** `torch/csrc/jit/serialization/import_source.cpp` (line 411)
**Severity:** None
**Description:** `static QualifiedName torch_classes_qualname(...)` is a const-like object initialized once. Thread-safe via magic statics, and only read afterwards.
**Status:** Safe.

### Issue 4: Static mutable registries in `pickler_helper.cpp`
**File:** `torch/csrc/jit/serialization/pickler_helper.cpp` (lines 98, 110)
**Severity:** Medium
**Description:** `GetBackendMetaAllowlist()` returns a reference to a `static std::unordered_set<c10::DeviceType>`. `GetBackendMetaSerialization()` returns a reference to a `static std::array<...>`. Both are mutable global registries -- callers can insert into the allowlist or assign to array slots. If this registration happens only during module initialization (single-threaded), the risk is low. However, if registration or queries can happen from multiple threads concurrently, this is a data race.
```cpp
std::unordered_set<c10::DeviceType>& GetBackendMetaAllowlist() {
  static std::unordered_set<c10::DeviceType> DeviceTypeAllowlist{...};
  return DeviceTypeAllowlist;
}
```
**Recommendation:** Protect with a mutex if registration can happen after startup, or document that registration must be done before multi-threaded operation begins.

### Issue 5: Static `const std::string kExportSuffix` in import_export_helpers.cpp
**File:** `torch/csrc/jit/serialization/import_export_helpers.cpp` (line 16)
**Severity:** None
**Description:** Immutable const string. Safe.
**Status:** Safe.

### Issue 6: `mobile_bytecode_generated.h` static arrays
**File:** `torch/csrc/jit/serialization/mobile_bytecode_generated.h` (multiple lines)
**Severity:** None
**Description:** All `static const` arrays/values in this generated flatbuffer header are immutable. Safe for concurrent access.
**Status:** Safe.

## Concurrency Notes

- `onnx.cpp` is purely functional: all functions take input parameters and produce output. No global state, no Python API usage. Fully safe.
- `pickle.cpp` and `pickle.h` are wrappers around `Pickler`/`Unpickler` which use per-instance state. Thread-safe as long as instances are not shared.
- `pickler.cpp`: The `static const size_t kSmallStr = 32` and `static constexpr uint8_t PROTOCOL_VERSION = 2` are constants. Safe.
- `import_read.cpp`: `static constexpr` values only. Safe.
- No Python C API calls in any of these files. All operate at the C++ layer.
