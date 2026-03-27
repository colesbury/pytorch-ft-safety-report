# Free-Threading Safety Report: api_include_torch_nativert

**Overall Risk: None**

## Files Analyzed
- `api/include/torch/nativert/ModelRunnerHandle.h`

## Analysis

No free-threading issues found.

This header declares the `ModelRunnerHandle` class, which is a pimpl wrapper around `ModelRunner`. It has a single `std::unique_ptr<ModelRunner> impl_` member. All state is instance-level. The class is explicitly non-copyable (copy constructor and copy assignment deleted). No Python C API usage. No static/global mutable state.
