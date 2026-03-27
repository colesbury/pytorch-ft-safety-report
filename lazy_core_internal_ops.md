# Free-Threading Safety Audit: lazy_core_internal_ops

## Overall Risk: LOW

## Files Audited
- `lazy/core/internal_ops/ltc_ops.h`

## Findings

### Issue 1: Global OpKindWrapper objects with lazy init (ltc_ops.h:34-48)

**Severity:** LOW
**Description:** Multiple `const OpKindWrapper` global objects are defined (e.g., `ltc_all_to_all`, `ltc_cast`, `ltc_device_data`, etc.). Each `OpKindWrapper` contains a `c10::once_flag` and uses `c10::call_once` for lazy initialization of the internal `OpKind` value. On first use from any thread, `call_once` ensures exactly one thread initializes the `op_kind_` member.
**Existing Protection:** `c10::call_once` provides thread-safe initialization. After initialization, only reads occur.
**Recommendation:** None. This is a correct lazy-init pattern.

### Issue 2: Namespace-scope const objects

**Severity:** NONE
**Description:** All `OpKindWrapper` objects are declared `const`. The mutable state within them (`op_kind_` and `once_`) is protected by `call_once`. This pattern is safe under free-threading.
**Existing Protection:** `const` + `call_once`.
**Recommendation:** None.

## Summary

This is a single header file containing global `OpKindWrapper` objects that use `c10::call_once` for thread-safe lazy initialization. No Python C API, no mutable shared state after initialization. Fully safe under free-threading.
