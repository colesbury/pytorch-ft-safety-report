# Free-Threading Audit: autograd_generated__part1

**Files:**
- `torch/csrc/autograd/generated/ADInplaceOrViewTypeEverything.cpp`
- `torch/csrc/autograd/generated/ADInplaceOrViewType_0.cpp`
- `torch/csrc/autograd/generated/ADInplaceOrViewType_1.cpp`
- `torch/csrc/autograd/generated/Functions.cpp`
- `torch/csrc/autograd/generated/Functions.h`
- `torch/csrc/autograd/generated/TraceTypeEverything.cpp`
- `torch/csrc/autograd/generated/TraceType_0.cpp`
- `torch/csrc/autograd/generated/TraceType_1.cpp`
- `torch/csrc/autograd/generated/TraceType_2.cpp`
- `torch/csrc/autograd/generated/TraceType_3.cpp`
- `torch/csrc/autograd/generated/TraceType_4.cpp`
- `torch/csrc/autograd/generated/VariableType.h`
- `torch/csrc/autograd/generated/VariableTypeEverything.cpp`
- `torch/csrc/autograd/generated/VariableType_0.cpp`
- `torch/csrc/autograd/generated/VariableType_1.cpp`

**Overall Risk:** Medium

## Issues

### 1. Non-atomic `static bool called` race in every `apply_with_saved` method (705 instances)
- **Category:** Static mutable C++ state in Python-facing functions / Lazy init pattern
- **Severity:** Medium
- **Confidence:** High
- **File:** `torch/csrc/autograd/generated/Functions.cpp`
- **Line(s):** 90, 169, 261, 348, 460, ... through 69061 (705 total instances, one per backward op class)
- **Description:** Every generated `apply_with_saved` method contains a `static bool called = false; if (!called) { called = true; ... bind_function(...); }` pattern. This is a non-atomic check-then-act on a plain `bool`. Under free-threading, if two threads execute the same backward op type's `apply_with_saved` concurrently, both can read `called` as `false` and both proceed to call `pyinterface->bind_function()`. The `bind_function` call goes into Python (`handle.attr("bind_function")`) which accesses Python objects. The data race on the `bool` itself is undefined behavior in C++, and the duplicate `bind_function` call may cause incorrect state in the Python compiler. Note: in practice, compiled autograd execution may currently be single-threaded, which would mitigate this, but the code is structurally racy.
- **Suggested Fix:** The fix must be in the code generator at `tools/autograd/gen_autograd_functions.py` line 139. Replace the plain `static bool called` with either `static std::atomic<bool> called{false}` with a `compare_exchange_strong` pattern, or use `std::call_once` / C++ magic static initialization to guarantee the binding happens exactly once. Example using `std::call_once`:
  ```cpp
  static std::once_flag flag;
  std::call_once(flag, [&]() {
      // compute schema and bind_function
  });
  ```

## Summary

The only free-threading issue found across these 15 generated files is the `static bool called` lazy-init race in `Functions.cpp`, repeated 705 times (once per backward op class). The ADInplaceOrView, TraceType, and VariableType files contain no mutable static state and no Python C API calls, making them safe. The `static c10::OperatorName` and `static ::std::optional<c10::OperatorHandle>` locals in VariableType files are safe because C++11 guarantees thread-safe initialization of function-scope statics, and they are only read after initialization. The fix for the `static bool called` issue needs to be applied in the code generator (`tools/autograd/gen_autograd_functions.py` line 139), not in the generated file directly.
