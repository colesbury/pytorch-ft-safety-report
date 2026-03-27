# Free-Threading Safety Audit: acc

**Overall Risk: Low**

## Files Reviewed
- `acc/Module.cpp`
- `acc/Module.h`

## Findings

### 1. Unprotected Check-Then-Act Registration Patterns
**Severity: Low**
**Location:** `acc/Module.cpp`, lines 121-139

```cpp
bool registerPythonPrivateUse1Hook(const py::object& hook) {
  if (at::isPrivateUse1HooksRegistered()) {
    return false;
  }
  hook.inc_ref();
  at::RegisterPrivateUse1HooksInterface(
      hook.cast<PrivateUse1HooksInterface*>());
  return true;
}

bool registerPythonPrivateUse1DeviceGuard(const py::object& guard) {
  if (c10::impl::hasDeviceGuardImpl(c10::DeviceType::PrivateUse1)) {
    return false;
  }
  guard.inc_ref();
  c10::impl::registerDeviceGuard(
      c10::DeviceType::PrivateUse1,
      guard.cast<c10::impl::DeviceGuardImplInterface*>());
  return true;
}
```

Both `registerPythonPrivateUse1Hook` and `registerPythonPrivateUse1DeviceGuard` perform a check-then-act pattern (check if registered, then register) without synchronization. Under free-threading, two threads could both see the "not registered" state and proceed to register concurrently. However, this is low risk because:
- Registration is typically done once during module initialization.
- The underlying `at::RegisterPrivateUse1HooksInterface` and `c10::impl::registerDeviceGuard` may have their own internal protections.
- Double registration would fail or be benign in typical usage patterns.

## Summary

The `acc` module is a thin pybind11 wrapper for PrivateUse1 accelerator hooks. The registration functions have a TOCTOU race, but this is unlikely to be triggered in practice since registration happens during module setup. All Python-facing functions go through pybind11 which provides its own GIL management layer. No mutable static/global PyObject* or other high-risk patterns were found.
