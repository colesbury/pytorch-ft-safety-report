# Free-Threading Audit: autograd__part1

**Files:**
- `torch/csrc/autograd/FunctionsManual.cpp`
- `torch/csrc/autograd/FunctionsManual.h`
- `torch/csrc/autograd/InferenceMode.h`
- `torch/csrc/autograd/TraceTypeManual.cpp`
- `torch/csrc/autograd/VariableTypeManual.cpp`
- `torch/csrc/autograd/VariableTypeUtils.h`
- `torch/csrc/autograd/anomaly_mode.cpp`
- `torch/csrc/autograd/anomaly_mode.h`
- `torch/csrc/autograd/autograd.cpp`
- `torch/csrc/autograd/autograd.h`
- `torch/csrc/autograd/autograd_meta.cpp`
- `torch/csrc/autograd/autograd_not_implemented_fallback.cpp`
- `torch/csrc/autograd/autograd_not_implemented_fallback.h`
- `torch/csrc/autograd/cpp_hook.cpp`
- `torch/csrc/autograd/cpp_hook.h`

**Overall Risk:** Medium

## Issues

### 1. Global `kAutogradFallbackMode` read/written without synchronization
- **Category:** Static/global mutable C++ state in Python-facing functions
- **Severity:** Low
- **Confidence:** High
- **File:** `torch/csrc/autograd/autograd_not_implemented_fallback.cpp`
- **Line(s):** 62, 66-73, 124, 129
- **Description:** `kAutogradFallbackMode` is a plain global `AutogradFallbackMode` enum value in an anonymous namespace. It is written by `setAutogradFallbackMode()` (exposed to Python as `torch._C._set_autograd_fallback_mode`) and read by `getAutogradFallbackMode()` and inside `basicAutogradNotImplementedFallbackImpl()` on the operator dispatch hot path. Under the GIL, concurrent Python calls to set and read were serialized. Without the GIL, a concurrent set and read is a data race (undefined behavior in C++). In practice the risk is low because this is typically set once at startup and the value is a small enum, but it is technically a data race.
- **Suggested Fix:** Make `kAutogradFallbackMode` a `std::atomic<AutogradFallbackMode>` with relaxed loads/stores, or convert to a thread-local if per-thread semantics are acceptable.

### 2. `AnomalyMode::_enabled` and `_check_nan` are non-atomic static bools with unsynchronized reads
- **Category:** Static/global mutable C++ state in Python-facing functions
- **Severity:** Low
- **Confidence:** Medium
- **File:** `torch/csrc/autograd/anomaly_mode.h` / `torch/csrc/autograd/anomaly_mode.cpp`
- **Line(s):** anomaly_mode.h:13-22, 25-26; anomaly_mode.cpp:9-10, 31, 38
- **Description:** `AnomalyMode::_enabled` and `AnomalyMode::_check_nan` are plain `static bool` members. They are read via `is_enabled()` and `should_check_nan()` from many threads (autograd engine, saved_variable, function.h, dynamo) without any synchronization. They are written by `set_enabled()` which is called both from `DetectAnomalyGuard` (which holds a mutex for its counter but not for the bool writes themselves) and from Python via `autograd/init.cpp`. This is technically a data race under C++ memory model. Note: this is a pre-existing C++ threading issue (the autograd engine is multi-threaded even with the GIL), but free-threading makes it more likely to be triggered from the Python side as well.
- **Suggested Fix:** Change `_enabled` and `_check_nan` to `std::atomic<bool>` with relaxed memory ordering. The `DetectAnomalyGuard` mutex already serializes the counter logic; the atomics would make the bool reads/writes well-defined.

## Summary

These files are predominantly pure C++ autograd logic (gradient formulas, variable type dispatch, view handling) with no Python C API usage. The two issues found are non-atomic global mutable state: `kAutogradFallbackMode` and `AnomalyMode` static bools. Both are low severity because they are rarely mutated in practice, but they are technically data races under C++ memory model and should use `std::atomic` for correctness.
