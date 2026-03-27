# Free-Threading Safety Audit: jit_runtime__part2

## Files Reviewed
- `jit/runtime/instruction.cpp`
- `jit/runtime/instruction.h`
- `jit/runtime/interpreter.cpp`
- `jit/runtime/interpreter.h`
- `jit/runtime/jit_exception.cpp`
- `jit/runtime/jit_exception.h`
- `jit/runtime/jit_trace.cpp`
- `jit/runtime/jit_trace.h`
- `jit/runtime/logging.cpp`
- `jit/runtime/logging.h`
- `jit/runtime/operator.cpp`
- `jit/runtime/operator.h`
- `jit/runtime/operator_options.h`
- `jit/runtime/print_handler.cpp`
- `jit/runtime/print_handler.h`

## Summary

This group contains the JIT interpreter, operator registry, logging, and related utilities. The operator registry (`OperatorRegistry`) is properly guarded by a mutex. The logging subsystem uses atomics for its global logger pointer. The main concerns are a mutable static in the operator registry's `getOperators` method, and the `PrintHandler` which uses `std::atomic` but stores a function pointer (which is safe). There are no Python C API interactions in these files.

## Issues

### Issue 1
- **Category**: Static mutable C++ state
- **Severity**: Medium
- **Confidence**: High
- **File**: `jit/runtime/operator.cpp`
- **Lines**: 140-145
- **Description**: `getOperators()` returns a `const` reference to either a map entry or a `static std::vector<std::shared_ptr<Operator>> empty`. The `empty` vector is function-local static, initialized once to an empty vector, and never modified -- it is safe. However, the method returns a reference to a vector inside `operators` (the map), and while the lock is held during the lookup, the returned reference escapes the lock scope. If another thread calls `registerOperator` or `registerPendingOperators` concurrently, the vector could be modified (elements pushed) while the caller iterates over the returned reference. This is a race condition.
- **Fix**: Return a copy of the vector instead of a reference, or document that callers must hold their own lock. Alternatively, have callers capture the returned reference into a local copy.

### Issue 2
- **Category**: Static mutable C++ state
- **Severity**: Medium
- **Confidence**: High
- **File**: `jit/runtime/operator.cpp`
- **Lines**: 120-135
- **Description**: `lookupByLiteral()` returns a `const std::shared_ptr<Operator>&` reference into the `operators_by_sig_literal` map. This reference escapes the lock scope. Concurrent modifications to the map (from `registerOperator` triggering `registerPendingOperators`) could invalidate the reference. Since `std::shared_ptr` is not trivially copyable, even reading the reference while another thread modifies the map is a data race.
- **Fix**: Return `std::shared_ptr<Operator>` by value instead of by reference.

### Issue 3
- **Category**: Static mutable C++ state
- **Severity**: Low
- **Confidence**: High
- **File**: `jit/runtime/operator.cpp`
- **Lines**: 191-262
- **Description**: `printerHasSpecialCaseFor` and `aliasAnalysisHasSpecialCaseFor` use `const static std::unordered_set<Symbol>` variables. These are initialized once (C++11 guarantee) and then only read. Safe.
- **Fix**: No fix needed.

### Issue 4
- **Category**: Static/global mutable state
- **Severity**: Low
- **Confidence**: High
- **File**: `jit/runtime/logging.cpp`
- **Lines**: 46-58
- **Description**: `global_logger` is `std::atomic<LoggerBase*>`, and `setLogger` uses a compare-exchange loop. This is thread-safe. However, the replaced `LoggerBase*` is returned to the caller but the old logger object is never deleted, which could be a memory leak (not a thread-safety issue per se). The `NoopLogger` created with `new` at line 46 is never freed, which is intentional for process-lifetime globals.
- **Fix**: No fix needed for thread-safety; the memory leak of the initial `NoopLogger` is acceptable for a global.

### Issue 5
- **Category**: Static mutable C++ state
- **Severity**: Low
- **Confidence**: High
- **File**: `jit/runtime/print_handler.cpp`
- **Lines**: 11
- **Description**: `print_handler` is `std::atomic<PrintHandler>` where `PrintHandler` is a function pointer. Loads and stores are atomic. This is thread-safe.
- **Fix**: No fix needed.

### Issue 6
- **Category**: Lazy initialization / mutable schema
- **Severity**: Low
- **Confidence**: Medium
- **File**: `jit/runtime/operator.h`
- **Lines**: 167-184
- **Description**: The `schema()` method on `JitOnlyOperator` lazily parses the schema string, mutating `op.schema_` (a mutable variant). This is guarded by nothing -- if two threads call `schema()` concurrently on the same `Operator` object, both may try to parse and assign to the variant simultaneously. In practice, operators are typically registered at startup and their schema is accessed after registration, but under free-threading this could race.
- **Fix**: Add `std::call_once` or a mutex to protect the lazy parsing of the schema. Alternatively, parse eagerly at registration time.
