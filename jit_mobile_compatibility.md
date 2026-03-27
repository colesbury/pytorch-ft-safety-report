# Free-Threading Safety Audit: jit_mobile_compatibility

## Files Audited
- `jit/mobile/compatibility/backport.cpp`
- `jit/mobile/compatibility/backport.h`
- `jit/mobile/compatibility/backport_manager.cpp`
- `jit/mobile/compatibility/backport_manager.h`
- `jit/mobile/compatibility/model_compatibility.cpp`
- `jit/mobile/compatibility/model_compatibility.h`
- `jit/mobile/compatibility/runtime_compatibility.cpp`
- `jit/mobile/compatibility/runtime_compatibility.h`

## Summary

This group implements model compatibility checking and version backporting for mobile models. It includes the `BackportManager` that manages a registry of backport functions, model introspection utilities to extract bytecode/operator versions, and a compatibility checker that compares model requirements against runtime capabilities. The code is pure C++ with no Python C API usage.

## Findings

### 1. Static `backportManager` in backport.cpp
- **File**: `jit/mobile/compatibility/backport.cpp`, line 13
- **Severity**: Low
- **Description**: `const static BackportManager backportManager` is a file-scope static. Its constructor registers backport functions into a static map (see finding #2). The `const` qualifier means the object itself is not modified after construction, but the constructor modifies shared state. Since this is a file-scope static, it is initialized during program startup (before `main`). Thread-safe for reads after initialization.

### 2. Static backport functions map in backport_manager.cpp
- **File**: `jit/mobile/compatibility/backport_manager.cpp`, lines 576-585
- **Severity**: Low
- **Description**: `BackportManager::bytecodeBackportFunctions()` returns a reference to a `static std::unordered_map`. The map is populated during the `BackportManager` constructor (called during static init of the `backportManager` object in backport.cpp). After static init, the map is only read via `hasBytecodeBackportFunction()` and `backport()`. Since all writes happen during static initialization and all reads happen at runtime, this is safe in practice.
- **Potential Issue**: If `BackportManager` were constructed dynamically after program start (e.g., from a dynamically loaded library), there could be races. But the current usage pattern (file-scope `const static`) makes this safe.

### 3. `_get_runtime_ops_and_info()` in runtime_compatibility.cpp
- **File**: `jit/mobile/compatibility/runtime_compatibility.cpp`, lines 31-63
- **Severity**: None
- **Description**: This function queries the dispatcher and JIT operator registry to build a map. It creates and returns a new map each call (no static state). The dispatcher itself uses appropriate internal synchronization. Thread-safe.

### 4. `_get_mobile_supported_types()` in runtime_compatibility.cpp
- **File**: `jit/mobile/compatibility/runtime_compatibility.cpp`, lines 73-86
- **Severity**: None
- **Description**: Reads from `DynamicTypeFactory::basePythonTypes()` and `TypeParser::getNonSimpleType()` / `TypeParser::getCustomType()`, all of which return references to const static collections. Creates and returns a new set each call. Thread-safe.

### 5. Model compatibility introspection functions (model_compatibility.cpp)
- **File**: `jit/mobile/compatibility/model_compatibility.cpp`
- **Severity**: None
- **Description**: All the `_get_model_*` functions create local `PyTorchStreamReader` instances and work with local state. They have no shared mutable state. Thread-safe as long as the input streams are not shared between threads.

### 6. `readArchive()` in model_compatibility.cpp
- **File**: `jit/mobile/compatibility/model_compatibility.cpp`, lines 26-57
- **Severity**: None
- **Description**: Creates local `CompilationUnit` and `mobile::CompilationUnit` instances. No shared state. Thread-safe.

### 7. `is_compatible()` function
- **File**: `jit/mobile/compatibility/model_compatibility.cpp`, lines 333-432
- **Severity**: None
- **Description**: A pure function that compares two info structs. No mutable state. Thread-safe.

## No Python C API Usage

None of the files in this group use the Python C API. All code operates at the C++/ATen layer.

## Overall Risk Assessment: LOW

The compatibility and backporting code is well-structured with minimal shared mutable state. The backport function registry is populated during static initialization and only read afterward. All model introspection functions use local state. No significant free-threading concerns.
