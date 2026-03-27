# Free-Threading Audit: autograd_utils

**Files:**
- `torch/csrc/autograd/utils/grad_layout_contract.h`
- `torch/csrc/autograd/utils/lambda_post_hook.h`
- `torch/csrc/autograd/utils/python_arg_parsing.h`
- `torch/csrc/autograd/utils/warnings.cpp`
- `torch/csrc/autograd/utils/warnings.h`
- `torch/csrc/autograd/utils/wrap_outputs.h`

**Note:** `python_arg_parsing.cpp` does not exist; only the header is present.

**Overall Risk:** None

## Issues

No free-threading issues found.

## Summary

These files contain no static or global mutable `PyObject*` state, no lazy-init patterns, and no shared mutable C++ state in Python-facing functions. `grad_layout_contract.h` and `lambda_post_hook.h` are pure C++ with no Python interaction. `python_arg_parsing.h` is a stateless inline helper. `DelayWarningHandler` in `warnings.cpp`/`warnings.h` already protects its internal state with a `std::mutex`. `wrap_outputs.h` only creates new Python objects via standard C API calls and returns new references, with no shared mutable state -- `PyTuple_SET_ITEM` is used exclusively on freshly allocated tuples before they are shared, which is safe.
