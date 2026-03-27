# Free-Threading Safety Audit: lazy_backend

## Overall Risk: LOW

## Files Audited
- `lazy/backend/backend_data.h`
- `lazy/backend/backend_device.cpp`
- `lazy/backend/backend_device.h`
- `lazy/backend/backend_interface.cpp`
- `lazy/backend/backend_interface.h`
- `lazy/backend/lowering_context.cpp`
- `lazy/backend/lowering_context.h`

## Findings

### Issue 1: Global backend registry (backend_interface.cpp:8)

**Severity:** LOW
**Description:** `std::atomic<const BackendImplInterface*> backend_impl_registry;` stores the registered backend implementation. `BackendRegistrar` stores to it, and `getBackend()` / `hasBackend()` load from it. The atomic ensures visibility.
**Existing Protection:** `std::atomic` store/load.
**Recommendation:** Safe as-is. The backend is registered once during initialization and then only read. The `TORCH_CHECK` in `getBackend()` will catch use-before-init.

### Issue 2: Static IrBuilder cache (backend_interface.cpp:28-30)

**Severity:** LOW
**Description:** `static const IrBuilder* builder = getBackend()->GetIrBuilder();` uses C++ thread-safe static initialization. After initialization, `builder` is a read-only pointer.
**Existing Protection:** Thread-safe C++ static init, then read-only.
**Recommendation:** None.

### Issue 3: BackendData::SetInfo (backend_data.h:40-43)

**Severity:** LOW
**Description:** `SetInfo` does a `std::swap` on a `std::shared_ptr<Info>` member. If multiple threads call `SetInfo` concurrently on the same `BackendData` object, this would be a data race. However, `BackendData` objects are typically not shared across threads in the lazy tensor design.
**Existing Protection:** Relies on single-owner semantics.
**Recommendation:** Acceptable under current usage patterns. If shared access becomes possible, add synchronization.

### Issue 4: Stateless functions

**Severity:** NONE
**Description:** `backend_device.cpp` functions (`atenDeviceToBackendDevice`, `backendDeviceToAtenDevice`, `GetBackendDevice`) are stateless -- they call into the backend interface (which is atomically registered) and perform value conversions. `lowering_context.cpp` is a simple constructor/accessor with no global state.
**Existing Protection:** Stateless.
**Recommendation:** None.

## Summary

The backend layer uses a properly atomic global registry for the backend implementation and thread-safe static initialization for the IR builder. The `LoweringContext` and `BackendDevice` classes are value types or simple wrappers with no shared mutable state. The only minor concern is `BackendData::SetInfo` which is not synchronized, but this follows the single-owner pattern. No Python C API is used.
