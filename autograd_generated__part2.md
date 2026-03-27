# Free-Threading Audit: autograd_generated__part2

**Files:**
- `torch/csrc/autograd/generated/VariableType_2.cpp`
- `torch/csrc/autograd/generated/VariableType_3.cpp`
- `torch/csrc/autograd/generated/VariableType_4.cpp`
- `torch/csrc/autograd/generated/ViewFuncs.cpp`
- `torch/csrc/autograd/generated/ViewFuncs.h`
- `torch/csrc/autograd/generated/python_enum_tag.cpp`
- `torch/csrc/autograd/generated/python_fft_functions.cpp`
- `torch/csrc/autograd/generated/python_functions.h`
- `torch/csrc/autograd/generated/python_functionsEverything.cpp`
- `torch/csrc/autograd/generated/python_functions_0.cpp`
- `torch/csrc/autograd/generated/python_functions_1.cpp`
- `torch/csrc/autograd/generated/python_functions_2.cpp`
- `torch/csrc/autograd/generated/python_functions_3.cpp`
- `torch/csrc/autograd/generated/python_functions_4.cpp`
- `torch/csrc/autograd/generated/python_linalg_functions.cpp`

**Overall Risk:** Low

## Issues

### 1. Global mutable `PyObject*` for FFT module reference
- **Category:** Static/global mutable `PyObject*`
- **Severity:** Low
- **Confidence:** Medium
- **File:** `torch/csrc/autograd/generated/python_fft_functions.cpp`
- **Line(s):** 119, 130
- **Description:** `static PyObject* THPFFTVariableFunctionsModule = NULL` is a file-scope global mutable pointer. It is written once in `initFFTFunctions()` (line 130) and then read from every `THPVariable_fft_*` function via `handle_torch_function`. Under free-threading, this is technically a data race (non-atomic write followed by non-atomic reads from other threads). In practice, the write happens during module import which is serialized by Python's import lock, and reads only occur after the module is importable, so exploitation is extremely unlikely. This is a generated file; the fix would be in the code generator template `tools/autograd/templates/python_fft_functions.cpp`.
- **Suggested Fix:** No immediate fix needed. If hardening is desired, the pointer could be stored in module state rather than a global, or made `std::atomic<PyObject*>` with relaxed store / acquire load. However, since Python's import serialization guarantees this is written before it can be read, the current code is safe in practice.

### 2. Global mutable `PyObject*` for linalg module reference
- **Category:** Static/global mutable `PyObject*`
- **Severity:** Low
- **Confidence:** Medium
- **File:** `torch/csrc/autograd/generated/python_linalg_functions.cpp`
- **Line(s):** 169, 180
- **Description:** Same pattern as issue 1: `static PyObject* THPLinalgVariableFunctionsModule = NULL` is written once during `initLinalgFunctions()` and read from every `THPVariable_linalg_*` function. Technically a non-atomic write/read pair, but practically safe due to import serialization. This is a generated file; the fix would be in `tools/autograd/templates/python_linalg_functions.cpp`.
- **Suggested Fix:** Same as issue 1 -- no immediate fix needed; use module state or `std::atomic` if hardening desired.

## Summary

These generated files are largely free of free-threading concerns. The VariableType files (2, 3, 4) are pure C++ autograd dispatch code with no Python C API usage and no problematic global mutable state. The ViewFuncs files are stateless C++ method implementations. The python_functions files (0-4, Everything) use only function-local `static PyTypeObject` and `static PythonArgParser` instances, both of which are safe (C++11 magic statics for thread-safe initialization, immutable after construction; type registration via `addClass`/`registerCppFunction` occurs only during serialized module init). The two low-severity issues are the `THPFFTVariableFunctionsModule` and `THPLinalgVariableFunctionsModule` global `PyObject*` pointers, which are write-once-read-many with writes serialized by the import lock, making them safe in practice.
