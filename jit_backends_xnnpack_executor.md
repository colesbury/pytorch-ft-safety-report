# Free-Threading Safety Audit: jit/backends/xnnpack/executor

**Overall Risk: Low**

## Files Reviewed
- `torch/csrc/jit/backends/xnnpack/executor/xnn_executor.h`

## Findings

### 1. XNNExecutor -- Per-instance mutable state without synchronization (Low)
**File:** `xnn_executor.h`, lines 12-66
```cpp
class XNNExecutor {
 private:
  std::unique_ptr<xnn_runtime, decltype(&xnn_delete_runtime)> runtime_{...};
  std::vector<uint32_t> input_ids_;
  std::vector<uint32_t> output_ids_;
  std::vector<xnn_external_value> externals_;
 public:
  template <typename T>
  bool set_inputs(std::vector<T*>& inputs, std::vector<T*>& outputs) {
    externals_.clear();
    ...
  }
  bool forward() {
    xnn_status status =
        xnn_setup_runtime(runtime_.get(), externals_.size(), externals_.data());
    ...
    status = xnn_invoke_runtime(runtime_.get());
    ...
  }
};
```
The `XNNExecutor` holds mutable per-instance state: `runtime_`, `input_ids_`, `output_ids_`, and `externals_`. The `set_inputs()` method mutates `externals_`, and `forward()` reads `externals_` and invokes the runtime.

If `set_inputs()` and `forward()` were called concurrently on the same `XNNExecutor` instance, there would be a data race on `externals_`. Similarly, concurrent `forward()` calls would race on `xnn_setup_runtime` + `xnn_invoke_runtime` since the runtime object is shared.

However, `XNNExecutor` is designed to be used sequentially: `set_inputs()` followed by `forward()`. It is wrapped in `XNNModelWrapper` (a `CustomClassHolder`) and accessed through the backend's `execute()` method. Concurrent `execute()` on the same handle is not an expected usage pattern.

**Severity: Low** -- Per-instance state with no synchronization. Safe under the expected single-threaded-per-handle usage pattern. Not safe for concurrent access to the same executor.

### 2. XNNExecutor::input_ids_ and output_ids_ -- Populated by friend class (Safe)
**File:** `xnn_executor.h`, line 65
```cpp
friend class XNNCompiler;
```
`input_ids_` and `output_ids_` are populated by `XNNCompiler::compileModel()` and then only read by `set_inputs()`. After compilation, these are effectively immutable. No race.

**Severity:** None

## Summary

`XNNExecutor` is a per-instance object with no static mutable state. Its mutable members (`externals_`, `runtime_`) are not protected by synchronization, but the intended usage pattern (sequential `set_inputs` then `forward`, one thread per handle) prevents races. This is a safe design for the expected usage.
