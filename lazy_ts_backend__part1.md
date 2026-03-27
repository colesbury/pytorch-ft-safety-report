# Free-Threading Safety Audit: lazy_ts_backend__part1

## Overall Risk: LOW-MEDIUM

## Files Audited
- `lazy/ts_backend/config.cpp`
- `lazy/ts_backend/config.h`
- `lazy/ts_backend/dynamic_ir.cpp`
- `lazy/ts_backend/dynamic_ir.h`
- `lazy/ts_backend/ir_builder.h`
- `lazy/ts_backend/tensor_aten_ops.cpp`
- `lazy/ts_backend/tensor_aten_ops.h`
- `lazy/ts_backend/ts_autograd_functions.cpp`
- `lazy/ts_backend/ts_autograd_functions.h`
- `lazy/ts_backend/ts_backend_impl.cpp`
- `lazy/ts_backend/ts_backend_impl.h`
- `lazy/ts_backend/ts_eager_fallback.cpp`
- `lazy/ts_backend/ts_eager_fallback.h`
- `lazy/ts_backend/ts_lowering_context.cpp`
- `lazy/ts_backend/ts_lowering_context.h`

## Findings

### Issue 1: Unprotected static map _eager_fallback_counters (ts_eager_fallback.cpp:145-146)

**Severity:** MEDIUM
**Description:** `static std::unordered_map<std::string, ::torch::lazy::Counter*> _eager_fallback_counters;` is read and written in `ltc_eager_fallback()` (line 177-180) without any synchronization. Under free-threading, multiple threads could simultaneously call `ltc_eager_fallback` for different operators, causing concurrent reads and writes to this map. `std::unordered_map` is not thread-safe for concurrent mutation.
**Existing Protection:** None.
**Recommendation:** Protect with a mutex, or use a concurrent map. Alternatively, since `Counter` objects are created per-operator and never deleted, a lock-free approach using `std::atomic` for the initial check and a mutex for insertion would suffice.

### Issue 2: Static local in force_eager_fallback (ts_eager_fallback.cpp:148-163)

**Severity:** LOW
**Description:** `static auto force_sym = c10::Symbol::fromQualString(std::string(force_str));` inside `force_eager_fallback()` is a function-local static that is initialized on the first call where `force_str` is non-empty. C++ guarantees thread-safe initialization of this static, but the value depends on `getLTCForceFallback()` which can change. Once initialized, the static won't update if the force fallback string changes. This is a logic issue rather than a thread-safety issue per se.
**Existing Protection:** C++ thread-safe static init (but the initialization is one-shot based on runtime state).
**Recommendation:** If dynamic changes to the force fallback string are expected, convert to a non-static variable computed each call.

### Issue 3: InitTorchScriptBackend static registrations (ts_backend_impl.cpp:272-281)

**Severity:** LOW
**Description:** `InitTorchScriptBackend()` creates static locals (`s_registrar`, `executor`) and calls registration functions. This is intended to be called once. The `static std::unique_ptr<BackendRegistrar> s_registrar` and `static LazyGraphExecutor* executor` are initialized once.
**Existing Protection:** C++ thread-safe static initialization. The registration functions (`RegisterTorchScriptLazyNativeFunctions`, etc.) are expected to be idempotent.
**Recommendation:** Safe as long as `InitTorchScriptBackend()` is called once during setup (it's called from `initLazyBindings` in Python).

### Issue 4: TSBackendImpl mutable members (ts_backend_impl.cpp:184-186)

**Severity:** LOW
**Description:** `default_device_type_` (a `shared_ptr`) and `default_device_ordinal_` are mutable members of `TSBackendImpl`. They are written by `SetDefaultDeviceType()` and `SetDefaultDeviceOrdinal()`, and read by other methods. The `TSBackendImpl` singleton is created once and shared. Concurrent read/write to `shared_ptr` is a data race.
**Existing Protection:** None explicit. These are configuration setters typically called during setup.
**Recommendation:** If configuration changes can happen concurrently with operation dispatch, protect with a mutex or use atomic operations.

### Issue 5: GetTSBackendImpl singleton (ts_backend_impl.cpp:267-269)

**Severity:** NONE
**Description:** `static TSBackendImpl* ts_backend_impl = new TSBackendImpl();` is a leaked singleton with thread-safe static init.
**Existing Protection:** C++ thread-safe static init.
**Recommendation:** None.

### Issue 6: register_ts_ltc_eager_fallback static library (ts_eager_fallback.cpp:197-204)

**Severity:** LOW
**Description:** `static auto m = MAKE_TORCH_LIBRARY_IMPL(_, Lazy);` creates a static library registration. `MAKE_TORCH_LIBRARY_IMPL` is designed for one-shot static registration. The `m.fallback(...)` call registers the fallback handler.
**Existing Protection:** Static init; this function is called once during `InitTorchScriptBackend`.
**Recommendation:** None.

### Issue 7: Stateless/value-type files

**Severity:** NONE
**Description:** `config.cpp/h` define flags (same pattern as core config). `dynamic_ir.cpp/h` define IR node classes with no static state. `ir_builder.h` defines `TorchScriptIrBuilder` (a stateless factory). `tensor_aten_ops.cpp/h` and `ts_autograd_functions.cpp/h` are stateless operation implementations. `ts_lowering_context.cpp/h` uses instance-level state only.
**Existing Protection:** Stateless or instance-level.
**Recommendation:** None.

## Summary

The primary concern in this group is the unprotected `_eager_fallback_counters` map in `ts_eager_fallback.cpp`, which can be written concurrently from multiple threads dispatching lazy operations. The `TSBackendImpl` has unprotected mutable configuration members, but these are typically only modified during setup. Other files are stateless or properly initialized via static singletons.
