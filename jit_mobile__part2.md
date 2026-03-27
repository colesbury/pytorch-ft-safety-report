# Free-Threading Safety Audit: jit_mobile__part2

## Files Audited
- `jit/mobile/interpreter.h`
- `jit/mobile/method.h`
- `jit/mobile/module.cpp`
- `jit/mobile/module.h`
- `jit/mobile/observer.cpp`
- `jit/mobile/observer.h`
- `jit/mobile/parse_bytecode.cpp`
- `jit/mobile/parse_bytecode.h`
- `jit/mobile/parse_operators.cpp`
- `jit/mobile/parse_operators.h`
- `jit/mobile/prim_ops_registery.cpp`
- `jit/mobile/prim_ops_registery.h`
- `jit/mobile/profiler_edge.cpp`
- `jit/mobile/profiler_edge.h`
- `jit/mobile/promoted_prim_ops.cpp`

## Summary

This group contains the mobile Module/Method runtime, operator parsing, primitive op registration, the edge profiler, and the bytecode parser. The code is pure C++ with no Python C API usage. The primary concerns are static mutable registries and unprotected global state used for profiling.

## Findings

### 1. Static mutable prim ops function table (prim_ops_registery.cpp)
- **File**: `jit/mobile/prim_ops_registery.cpp`, lines 5-10
- **Severity**: Medium
- **Description**: `primOpsFnTable()` returns a reference to a `static std::unordered_map<std::string, std::function<void(Stack&)>>`. Three functions access it without synchronization:
  - `registerPrimOpsFunction()` writes to the map
  - `hasPrimOpsFn()` reads the map
  - `getPrimOpsFn()` reads and returns a reference into the map

  Registration happens at static initialization time via `prim_op_fn_register` constructors in `promoted_prim_ops.cpp` (the `op_reg` array). The reads happen at runtime during model execution. Under free-threading, if a library is dynamically loaded while another thread is executing, the read and write could race.
- **Current GIL Dependence**: Static init happens before Python execution begins, and the GIL would serialize any dynamic loading. Without the GIL, dynamic library loading during execution could cause races.
- **Recommendation**: If all registration is guaranteed to complete before any reads (typical for static init), this is safe in practice. Otherwise, protect with a read-write lock or use a lock-free map.

### 2. Static `op_reg` array in promoted_prim_ops.cpp (safe)
- **File**: `jit/mobile/promoted_prim_ops.cpp`, lines 235-260
- **Severity**: None
- **Description**: `op_reg` is a `static const std::array<mobile::prim_op_fn_register, 16>`. The constructors call `registerPrimOpsFunction` during static initialization. Once construction is complete, `op_reg` itself is never modified. The concern is with `primOpsFnTable` (addressed above), not this array.

### 3. Thread-local edge profiler pointer (profiler_edge.cpp)
- **File**: `jit/mobile/profiler_edge.cpp`, line 10
- **Severity**: None
- **Description**: `tls_edge_profiler` is `thread_local KinetoEdgeCPUProfiler*`. Each thread has its own pointer. This is inherently safe for free-threading.

### 4. `std::rand()` usage in module.cpp (Method::run)
- **File**: `jit/mobile/module.cpp`, line 240
- **Severity**: Low
- **Description**: `std::rand()` is used to generate an `instance_key` for the observer. Same issue as in import.cpp - `std::rand()` is not thread-safe.
- **Recommendation**: Use a thread-safe RNG.

### 5. `observerConfig()` global state (observer.cpp/observer.h)
- **File**: `jit/mobile/observer.cpp`, lines 5-8; `jit/mobile/observer.h`, lines 97-108
- **Severity**: Low
- **Description**: (Duplicate with part1 analysis.) `observerConfig()` returns a static singleton. `setModuleObserver`/`getModuleObserver` are not synchronized. In typical usage the observer is configured once at startup, so this is safe in practice.
- **Recommendation**: No change needed if configuration-before-use pattern is maintained. Otherwise add synchronization.

### 6. `Code::initialized` flag (code.h, function.cpp)
- **File**: `jit/mobile/code.h`, line 31; `jit/mobile/function.cpp`, lines 64-93
- **Severity**: Medium
- **Description**: `Function::initialize_operators()` checks `code_.initialized` and if false, populates `code_.operators_` and sets `code_.initialized = true`. This is a classic check-then-act pattern without synchronization. If two threads call `run()` on the same `Function` concurrently, both could enter `initialize_operators()` simultaneously, causing a data race on `code_.operators_` (a `std::vector` being resized/written by both threads).
- **Current GIL Dependence**: Under GIL, only one thread executes Python code at a time, serializing access.
- **Recommendation**: Use `std::call_once` or an `std::atomic<bool>` with a mutex for the initialization path. Alternatively, ensure operators are always initialized during deserialization (before the module is shared across threads).

### 7. OpCodeCache in parse_bytecode.cpp (safe)
- **File**: `jit/mobile/parse_bytecode.cpp`, lines 37-66
- **Severity**: None
- **Description**: `OpCodeCache` is allocated as a local variable inside `parseInstructions()`. Each deserialization call gets its own instance. No shared state.

## No Python C API Usage

None of the files in this group use the Python C API. All concerns are C++-level static/global mutable state issues.

## Overall Risk Assessment: LOW-MEDIUM

The primary risks are the lazy operator initialization pattern in `Function::initialize_operators()` and the unprotected prim ops registry. Both follow patterns where writes happen early (static init or first model load) and reads happen at runtime. The risk is real but limited to scenarios involving concurrent first-time initialization of the same Function object or dynamic library loading during execution.
