# Free-Threading Safety Audit: jit/frontend (Part 3)

## Overall Risk: **Moderate**

Most files in this set are pure C++ JIT infrastructure (graph manipulation, type parsing, schema matching) that do not interact with the Python C API. The primary concerns are mutable static/global state in `tracer.cpp`, `strtod.cpp`, `schema_type_parser.cpp`, and `sugared_value.cpp`, plus function-pointer globals used for callbacks.

---

## Files Reviewed

### 1. `jit/frontend/schema_matching.cpp` / `.h`
**Risk: Low**

Pure C++ logic for matching function call arguments to operator schemas. No Python C API usage, no mutable statics, no global state. All functions operate on parameters passed in. Safe.

---

### 2. `jit/frontend/schema_type_parser.cpp` / `.h`
**Risk: Low**

- **Lines 47-55 (schema_type_parser.cpp): Global opaque types registry with mutex** -- The `getOpaqueTypes()` static `unordered_set` and `getOpaqueTypesMutex()` are properly synchronized. `registerOpaqueType`, `unregisterOpaqueType`, and `isRegisteredOpaqueType` all acquire the lock before accessing the set. This is correct for free-threading.

- **Lines 80-111 (schema_type_parser.cpp): Static local `type_map` in `parseBaseType()`** -- This is a `static std::unordered_map` initialized once (const-after-init). Since C++11 guarantees thread-safe initialization of function-local statics, and the map is never mutated after construction, this is safe.

- **Lines 221-222 (schema_type_parser.cpp): Static local `type_map` in `parseTensorDType()`** -- Same pattern as above: const-after-init static map. Safe.

No Python C API usage. No issues found.

---

### 3. `jit/frontend/script_type_parser.cpp` / `.h`
**Risk: Low**

Pure C++ type parsing from the TorchScript AST. No Python C API, no mutable statics, no global state. The `string_to_type_lut()` call at line 24 returns a const reference to a static map created via `c10::DefaultTypeFactory::basePythonTypes()`, which is const-after-init. Safe.

---

### 4. `jit/frontend/source_range.cpp` / `.h`
**Risk: Low**

`StringCordView` and `SourceRange` are value types with no shared mutable state. All operations are const or operate on local data. No Python C API. Safe.

---

### 5. `jit/frontend/source_ref.h`
**Risk: Low**

Simple wrapper holding a `shared_ptr<Source>`. No mutable statics, no Python C API. Safe.

---

### 6. `jit/frontend/strtod.cpp` / `.h`
**Risk: Low (with caveat)**

- **Lines 31-33 / 36-38 (strtod.cpp): Static locale objects** -- On MSVC, `_create_locale` creates a static `_locale_t`. On POSIX, `newlocale` creates a static `locale_t`. Both are initialized once via function-local statics (thread-safe init guaranteed by C++11) and are then used read-only by `_strtod_l` / `strtod_l`. The locale object itself is not mutated by these calls. Safe.

---

### 7. `jit/frontend/sugared_value.cpp` / `.h`
**Risk: Low**

- **Lines 32-44 (sugared_value.cpp): `builtin_cast_method_to_scalar_type()`** -- Static local `unordered_map`, const-after-init. Safe.

- **Lines 115-157 (sugared_value.cpp): `builtin_properties`** -- Static local `PropertiesLookup` map, const-after-init. Safe.

- **Line 288 (sugared_value.cpp): `make_simple_value` static lambda** -- This is a stateless lambda stored in a static local. No captures, no mutation. Safe.

No Python C API usage in this file. All logic operates on JIT `Value*`, `Graph`, and type objects.

---

### 8. `jit/frontend/tracer.cpp` / `.h`
**Risk: Moderate**

This is the file with the most concerns.

#### 8a. Thread-local tracing state (line 58)
```cpp
static thread_local std::shared_ptr<TracingState> tracing_state;
```
Being `thread_local`, this is inherently per-thread and safe from data races. No issue.

#### 8b. Global atomic: `tracer_state_warn_mode` (line 61)
```cpp
static std::atomic<bool> tracer_state_warn_mode{true};
```
Properly atomic. Safe.

#### 8c. Thread-local `ArgumentStash` (line 1002)
```cpp
thread_local ArgumentStash ArgumentStash::stash;
```
Thread-local, safe.

#### 8d. ISSUE: `record_source_location` function pointer (lines 1052-1060)
```cpp
static void defaultRecordSourceLocation(Node* n) {}
static std::atomic<decltype(&defaultRecordSourceLocation)>
    record_source_location(defaultRecordSourceLocation);
```
The atomic stores/loads are safe for the pointer itself. However, `setRecordSourceLocation` is typically called from Python initialization to install a Python-aware callback. Under free-threading, if one thread is calling `recordSourceLocation` while another thread calls `setRecordSourceLocation`, the atomic load ensures the pointer is consistent, but there is a potential issue if the old function is being unloaded or the callback references Python state that is being torn down. In practice, this is set once during module init and not changed thereafter, so the real-world risk is low.

**Severity: Low**
**Recommendation: None needed** -- the atomic is sufficient for the once-at-init pattern.

#### 8e. ISSUE: `python_callstack_fn` function pointer (lines 1062-1072)
```cpp
static std::atomic<decltype(&defaultPythonCallstack)> python_callstack_fn(
    defaultPythonCallstack);
```
Same pattern as `record_source_location`. Set once during init, accessed atomically. Safe for the same reasons.

**Severity: Low**

#### 8f. ISSUE: `warn_callback` function pointer (line 1077)
```cpp
static std::atomic<warn_fn_type> warn_callback{defaultWarn};
```
Same atomic function pointer pattern. Set once during init via `setWarn()`. Safe.

**Severity: Low**

#### 8g. Const global string literals (lines 1079-1098)
```cpp
const char* WARN_PYTHON_DATAFLOW = ...;
const char* WARN_CONSTRUCTOR = ...;
// etc.
```
These are effectively constant string pointers. However, they are declared as non-const `const char*` pointers (const data, but the pointer itself is mutable). In theory a thread could reassign the pointer, but no code does so. The lack of `constexpr` or making the pointer itself `const` (`const char* const`) is a minor style issue but not a practical concern.

**Severity: Negligible**

---

## Summary of Issues

| # | File | Lines | Issue | Severity | Category |
|---|------|-------|-------|----------|----------|
| 1 | tracer.cpp | 1052-1060 | Atomic function pointer for `record_source_location` -- safe as-is due to set-once-at-init pattern | Low | Static mutable C++ state |
| 2 | tracer.cpp | 1065-1072 | Atomic function pointer for `python_callstack_fn` -- safe as-is | Low | Static mutable C++ state |
| 3 | tracer.cpp | 1077 | Atomic function pointer for `warn_callback` -- safe as-is | Low | Static mutable C++ state |
| 4 | tracer.cpp | 1079-1098 | Global `const char*` pointers (non-const pointer to const data) -- not a practical issue | Negligible | Static mutable C++ state |

## Positive Observations

- The opaque types registry in `schema_type_parser.cpp` is properly mutex-protected.
- All static local maps (type_map, builtin_properties, builtin_cast_method_to_scalar_type) follow the safe const-after-init pattern.
- The tracer correctly uses `thread_local` for `TracingState` and `ArgumentStash`, which naturally avoids cross-thread data races.
- The atomic function-pointer callbacks in `tracer.cpp` are a reasonable design for the set-once-at-init pattern.
- None of these files directly use the Python C API (no `PyObject*`, no `PyGILState_Ensure`, no borrowed references from Python containers).
