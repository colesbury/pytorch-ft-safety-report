# Free-Threading Audit: cuda_shared

**Files:**
- `torch/csrc/cuda/shared/cudart.cpp`
- `torch/csrc/cuda/shared/cudnn.cpp`
- `torch/csrc/cuda/shared/cusparselt.cpp`
- `torch/csrc/cuda/shared/nvtx.cpp`

**Overall Risk:** Low

## Issues

### 1. No free-threading issues in `cudart.cpp`
- **Category:** N/A
- **Severity:** N/A
- **Confidence:** High
- **File:** `torch/csrc/cuda/shared/cudart.cpp`
- **Description:** `initCudartBindings` registers pybind11 bindings for CUDA runtime functions (`cudaProfilerStart`, `cudaHostRegister`, `cudaMemGetInfo`, etc.). All registered lambdas release the GIL before calling CUDA APIs using `py::gil_scoped_release`, which is correct. There is no mutable global state, no Python object manipulation outside the GIL, and no lazy init patterns.
- **Suggested Fix:** None needed.

### 2. No free-threading issues in `cudnn.cpp`
- **Category:** N/A
- **Severity:** N/A
- **Confidence:** High
- **File:** `torch/csrc/cuda/shared/cudnn.cpp`
- **Description:** `initCudnnBindings` registers pybind11 bindings for cuDNN/MIOpen version query functions. The functions (`getCompileVersion`, `getRuntimeVersion`, `getVersionInt`) read compile-time constants or call thread-safe library version query APIs. No mutable state, no Python object manipulation.
- **Suggested Fix:** None needed.

### 3. No free-threading issues in `cusparselt.cpp`
- **Category:** N/A
- **Severity:** N/A
- **Confidence:** High
- **File:** `torch/csrc/cuda/shared/cusparselt.cpp`
- **Description:** `initCusparseltBindings` registers pybind11 bindings for cuSPARSELt version and matrix multiplication search. The `getVersionInt` function returns a compile-time constant. `mmSearch` calls into `at::native::_cslt_sparse_mm_impl`, which operates on tensor arguments. No mutable global state.
- **Suggested Fix:** None needed.

### 4. No free-threading issues in `nvtx.cpp`
- **Category:** N/A
- **Severity:** N/A
- **Confidence:** High
- **File:** `torch/csrc/cuda/shared/nvtx.cpp`
- **Description:** `initNvtxBindings` registers pybind11 bindings for NVTX range/mark APIs. The NVTX functions (`nvtxRangePushA`, `nvtxRangePop`, `nvtxRangeStartA`, `nvtxRangeEnd`, `nvtxMarkA`) are thread-safe by design (NVTX is designed for use from multiple threads). The `device_nvtxRangeStart` and `device_nvtxRangeEnd` helper functions use `cudaLaunchHostFunc` to schedule NVTX operations on CUDA streams, which is also thread-safe. The `RangeHandle` struct is allocated per-call and freed in the callback, so there is no shared mutable state.
- **Suggested Fix:** None needed.
