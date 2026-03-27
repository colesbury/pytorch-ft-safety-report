# Free-Threading Safety Audit: jit_mobile__part3

## Files Audited
- `jit/mobile/promoted_prim_ops.h`
- `jit/mobile/quantization.cpp`
- `jit/mobile/quantization.h`
- `jit/mobile/register_ops_common_utils.cpp`
- `jit/mobile/register_ops_common_utils.h`
- `jit/mobile/type_parser.cpp`
- `jit/mobile/type_parser.h`
- `jit/mobile/upgrader_mobile.cpp`
- `jit/mobile/upgrader_mobile.h`

## Summary

This group contains helper utilities for mobile op registration, quantization, the mobile type parser, and the auto-generated upgrader bytecode. The code is pure C++ with no Python C API usage. Most of the code is either stateless utility functions or uses local/instance state only.

## Findings

### 1. Static local maps in upgrader_mobile.cpp
- **File**: `jit/mobile/upgrader_mobile.cpp`, lines 20-89 (getOperatorVersionMapForMobile)
- **Severity**: None
- **Description**: `getOperatorVersionMapForMobile()` constructs and returns a `static std::unordered_map`. However, it is constructed fresh each call (returns by value, not by reference). The `static` keyword here applies to the local initializer, and since the map is `const`-qualified at the call site and returned by value, there is no shared mutable state concern. Actually, examining the code more carefully: it is `static` inside the function and returned by value each time. The static local initialization is thread-safe (C++11 guarantee), and the returned copy is per-call. This is safe but potentially wasteful.

### 2. Static upgrader bytecode list in upgrader_mobile.cpp
- **File**: `jit/mobile/upgrader_mobile.cpp`, line 92+ (getUpgraderBytecodeList)
- **Severity**: Medium
- **Description**: `getUpgraderBytecodeList()` uses a static local lambda that generates the upgrader function list. The lambda internally calls `Function::registerFunc()` which uses a static map (as noted in part1 finding #1). The `getUpgraderBytecodeList()` itself likely uses a `static` vector that is lazily initialized. Multiple threads calling this function concurrently during first model load could race on:
  1. The static local initialization of the vector (safe due to C++11 guarantee)
  2. The internal `Function::registerFunc()` calls that mutate the static `upgrader_function_holder` (unsafe - same issue as part1 finding #1)
- **Recommendation**: The `Function::registerFunc` static map needs synchronization (see part1 finding #1).

### 3. Static locals in TypeParser (type_parser.cpp)
- **File**: `jit/mobile/type_parser.cpp`, lines 68-78
- **Severity**: None
- **Description**: `getNonSimpleType()` and `getCustomType()` return references to `static std::unordered_set<std::string>` instances. These are initialized once (C++11 static local guarantee) and never modified afterward. Thread-safe.

### 4. Static local in TypeParser::parseList (type_parser.cpp)
- **File**: `jit/mobile/type_parser.cpp`, line 44
- **Severity**: None
- **Description**: `static const c10::QualifiedName classPrefix` is a const static local. Initialized once, never modified. Thread-safe.

### 5. Static constexpr in type_parser.cpp (safe)
- **File**: `jit/mobile/type_parser.cpp`, lines 17-18
- **Severity**: None
- **Description**: `kTypeTorchbindCustomClass` and `kTypeNamedTuple` are `static constexpr const char*`. Immutable. Thread-safe.

### 6. Static constexpr in TypeParser::parseTorchbindClassType (safe)
- **File**: `jit/mobile/type_parser.cpp`, line 251
- **Severity**: None
- **Description**: `static constexpr std::array<const char*, 4> expected_atoms` is constexpr. Immutable. Thread-safe.

### 7. PTQQuanizationHelper (quantization.cpp)
- **File**: `jit/mobile/quantization.cpp`
- **Severity**: None
- **Description**: `quantize_dynamic` takes a mutable reference to a `Module` and mutates it in place. The function itself has no static/global state. Thread safety is the caller's responsibility (don't share a Module being quantized across threads).

### 8. Utility functions (register_ops_common_utils.cpp)
- **File**: `jit/mobile/register_ops_common_utils.cpp`
- **Severity**: None
- **Description**: `normalizeIndex` and `tensorToListRecursive` are pure functions with no static/global state. Thread-safe.

### 9. `to_dispatch` static function (register_ops_common_utils.h)
- **File**: `jit/mobile/register_ops_common_utils.h`, lines 17-35
- **Severity**: None
- **Description**: A stateless utility function. Thread-safe.

## No Python C API Usage

None of the files in this group use the Python C API. All concerns are limited to C++ static state patterns.

## Overall Risk Assessment: LOW

The only real concern is the transitive dependency on `Function::registerFunc`'s unprotected static map, which is exercised through `getUpgraderBytecodeList()`. The type parser and other utilities use safe patterns (const statics, local state, or stateless functions).
