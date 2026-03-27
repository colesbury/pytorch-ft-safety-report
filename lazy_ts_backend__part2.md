# Free-Threading Safety Audit: lazy_ts_backend__part2

## Overall Risk: LOW

## Files Audited
- `lazy/ts_backend/ts_native_functions.cpp`
- `lazy/ts_backend/ts_node.cpp`
- `lazy/ts_backend/ts_node.h`
- `lazy/ts_backend/ts_node_lowering.cpp`
- `lazy/ts_backend/ts_node_lowering.h`

## Findings

### Issue 1: Static env check in ts_node.cpp (lines 7-8)

**Severity:** NONE
**Description:** `static const auto LTC_ENABLE_SOURCE_INFO = c10::utils::has_env("LTC_ENABLE_SOURCE_INFO");` is a `const` static with thread-safe initialization. After initialization, it is read-only.
**Existing Protection:** C++ thread-safe static init, then `const`.
**Recommendation:** None.

### Issue 2: Global const OpKind in ts_node.h (line 70)

**Severity:** NONE
**Description:** `const OpKind tensor_list_opkind = OpKind::Get("lazy_tensors::tensor_list");` is a namespace-scope constant. `OpKind::Get` calls `c10::Symbol::fromQualString` which accesses the global symbol table (which is thread-safe in c10).
**Existing Protection:** `const` global, thread-safe symbol table.
**Recommendation:** None.

### Issue 3: Static sync flag in ts_native_functions.cpp (line 73)

**Severity:** LOW
**Description:** `static bool sync_update = FLAGS_torch_lazy_ts_tensor_update_sync;` captures the flag value once. This is a thread-safe static init of a `bool`. After init, it is read-only.
**Existing Protection:** C++ thread-safe static init.
**Recommendation:** None. Note that changes to the flag after initialization will not be reflected.

### Issue 4: Stateless functions

**Severity:** NONE
**Description:** Most functions in `ts_native_functions.cpp` (`LazyNativeFunctions::clone`, `_copy_from`, `_to_copy`, `empty_symint`, etc.) are stateless -- they operate on their arguments and delegate to the lazy tensor infrastructure. `ts_node_lowering.cpp` contains stateless lowering functions (`LowerBuiltin`, `LowerTSBuiltin`, `GenerateClone`). No mutable global/static state.
**Existing Protection:** Stateless.
**Recommendation:** None.

### Issue 5: TsNode class (ts_node.h/cpp)

**Severity:** NONE
**Description:** `TsNode` is a value-like IR node class. Its members (`shape_hash_`, `dag_hash_`) are set during construction and then only read. The `getPythonStacktrace()` method calls into the Python frame utilities (analyzed in lazy_python report).
**Existing Protection:** Immutable after construction.
**Recommendation:** None.

## Summary

These files are primarily stateless function implementations and IR node class definitions. The `TsNode` class is immutable after construction. The native functions delegate to the core lazy tensor infrastructure. One flag is cached in a static bool, which is safe but means the value is frozen at first use. No Python C API is used directly (Python stack traces are obtained through the function pointer mechanism analyzed in other reports). No significant free-threading concerns.
