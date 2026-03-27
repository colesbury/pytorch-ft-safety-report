# Free-Threading Safety Audit: jit_mobile_model_tracer

## Files Audited
- `jit/mobile/model_tracer/BuildFeatureTracer.cpp`
- `jit/mobile/model_tracer/BuildFeatureTracer.h`
- `jit/mobile/model_tracer/CustomClassTracer.cpp`
- `jit/mobile/model_tracer/CustomClassTracer.h`
- `jit/mobile/model_tracer/KernelDTypeTracer.cpp`
- `jit/mobile/model_tracer/KernelDTypeTracer.h`
- `jit/mobile/model_tracer/MobileModelRunner.cpp`
- `jit/mobile/model_tracer/MobileModelRunner.h`
- `jit/mobile/model_tracer/OperatorCallTracer.cpp`
- `jit/mobile/model_tracer/OperatorCallTracer.h`
- `jit/mobile/model_tracer/TensorUtils.cpp`
- `jit/mobile/model_tracer/TensorUtils.h`
- `jit/mobile/model_tracer/TracerRunner.cpp`
- `jit/mobile/model_tracer/TracerRunner.h`
- `jit/mobile/model_tracer/tracer.cpp`

## Summary

This group implements the mobile model tracing infrastructure, used to determine which operators, dtypes, custom classes, and build features a model uses. The tracers attach global callbacks via the `RecordFunction` mechanism and collect data into static synchronized containers. The `tracer.cpp` file is a standalone binary (`main()`). The code is pure C++ with no Python C API usage.

## Findings

### 1. Tracers use `c10::Synchronized` containers (safe)
- **Files**: `BuildFeatureTracer.cpp`, `CustomClassTracer.cpp`, `KernelDTypeTracer.cpp`, `OperatorCallTracer.cpp`
- **Severity**: None
- **Description**: All four tracer classes store their collected data in `static c10::Synchronized<T>` containers:
  - `BuildFeatureTracer::getBuildFeatures()` - `c10::Synchronized<build_feature_type>`
  - `CustomClassTracer::getLoadedClasses()` - `c10::Synchronized<custom_classes_type>`
  - `KernelDTypeTracer::getCalledKernelTags()` - `c10::Synchronized<kernel_tags_type>`
  - `OperatorCallTracer::getCalledOperators()` - `c10::Synchronized<std::set<std::string>>`

  All access to these containers goes through `withLock()`, which provides proper synchronization. This is a good pattern for free-threading safety.

### 2. Tracer callback registration via `at::addGlobalCallback` (safe for intended use)
- **Files**: All tracer constructors
- **Severity**: None
- **Description**: The tracers register global callbacks using `at::addGlobalCallback()`. The `RecordFunction` callback infrastructure has its own internal synchronization. The tracers' destructors call `at::removeCallback()`. This follows the intended API usage pattern.

### 3. Documentation notes tracers are not thread-safe (acknowledged)
- **Files**: All tracer headers
- **Severity**: Low (informational)
- **Description**: Each tracer header includes a note: "This class is not thread safe or re-entrant, and should not be used across multiple threads of execution." This refers to the tracer lifecycle (construction/destruction), not the data collection. The data collection is synchronized via `c10::Synchronized`. The warning is about not constructing/destroying tracers from multiple threads concurrently, which is appropriate.

### 4. `MobileModelRunner` uses shared_ptr to Module
- **File**: `jit/mobile/model_tracer/MobileModelRunner.h`, `MobileModelRunner.cpp`
- **Severity**: None
- **Description**: `MobileModelRunner` holds a `std::shared_ptr<Module>`. The Module is only used by the runner's own methods, and the runner is used as a local object within `TracerRunner.cpp`. No cross-thread sharing.

### 5. Static `gpu_metal_operators` vector in TracerRunner.cpp (safe)
- **File**: `jit/mobile/model_tracer/TracerRunner.cpp`, lines 21-41
- **Severity**: None
- **Description**: `gpu_metal_operators` is a `const std::vector<std::string>`. Immutable after initialization. Thread-safe.

### 6. Static `always_included_traced_ops` in TracerRunner.h (safe)
- **File**: `jit/mobile/model_tracer/TracerRunner.h`, lines 14-18
- **Severity**: None
- **Description**: `always_included_traced_ops` is `const`. Immutable. Thread-safe.

### 7. `tracer.cpp` main() function
- **File**: `jit/mobile/model_tracer/tracer.cpp`
- **Severity**: None
- **Description**: This is a standalone binary entry point. It runs single-threaded. No free-threading concerns.

### 8. `for_each_tensor_in_ivalue` (TensorUtils.cpp)
- **File**: `jit/mobile/model_tracer/TensorUtils.cpp`
- **Severity**: None
- **Description**: A pure recursive traversal function with no static state. Thread-safe.

## No Python C API Usage

None of the files in this group use the Python C API.

## Overall Risk Assessment: LOW

The model tracer code is well-designed for thread safety. All shared mutable state is protected by `c10::Synchronized`. The tracer classes themselves are documented as not thread-safe for lifecycle management, but this is appropriate for their intended use pattern (created and destroyed in a single-threaded context, with data collection callbacks safely synchronized). No changes needed for free-threading.
