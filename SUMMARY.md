# Free-Threading Safety Audit Summary — `torch/csrc/`

## Audit Scope

- **Files audited:** 1795 `.cpp` and `.h` files in `torch/csrc/`
- **Report groups:** 201/201 completed
- **Total issues found:** 350
  - High: 15
  - Medium: 60
  - Low: 71

## Report Risk Distribution

- **High:** 10 reports
- **Medium:** 32 reports
- **Low:** 71 reports
- **None:** 55 reports
- **Unknown:** 33 reports

## High Severity Issues (15 total)

### 1. Lazy-Initialized Static `PyObject*` Without Synchronization
- **Severity:** High | **Confidence:** Unknown
- **Location:** `torch/csrc/`
- **Report:** [fx](fx.md)

### 2. Static `worker_pids` map accessed without synchronization from Python methods
- **Severity:** High | **Confidence:** High
- **Location:** `torch/csrc/DataLoader.cpp:121, 129-172, 186-201, 213-219`
- **Report:** [toplevel__part1](toplevel__part1.md)
- **Description:** The file-static `std::map<int64_t, std::set<pid_t>> worker_pids` is read and mutated by three different Python-callable functions (`THPModule_errorIfAnyWorkerFails`, `THPModule_setWorkerPIDs`, `THPModule_removeWorkerPIDs`) without any mutex protection. Under the GIL, these calls were serialized. Und
- **Suggested Fix:** Add a `std::mutex` guarding all accesses to `worker_pids`, or use `Py_BEGIN_CRITICAL_SECTION` on a shared lock object.

### 3. `LogAPIUsageOnceFromPython` uses unprotected static `std::unordered_set`
- **Severity:** High | **Confidence:** High
- **Location:** `torch/csrc/Module.cpp:2125-2131`
- **Report:** [toplevel__part2](toplevel__part2.md)
- **Description:** `LogAPIUsageOnceFromPython` contains a `static std::unordered_set<std::string> seen` that is read and written without synchronization. The comment says "Guaranteed to be invoked from Python under GIL, no locking on map needed" -- this guarantee no longer holds under free-threading. Concurrent calls 
- **Suggested Fix:** Protect with a `std::mutex`, or use a concurrent set. Alternatively, use `std::call_once` per event string, though that's impractical for dynamic keys. A simple `static std::mutex` guarding the set is

### 4. Global mutable `PyObject*` variables set during module init
- **Severity:** High | **Confidence:** High
- **Location:** `torch/csrc/autograd/init.cpp:110, 119, 126-127, 143`
- **Report:** [autograd__part2](autograd__part2.md)
- **Description:** `THPAutograd_initExtension` sets four global `PyObject*` variables (`THPVariableClass`, `THPFunctionClass`, `THPGradientEdgeClass`, `ParameterClass`) without synchronization. Under free-threading, if two threads import the module concurrently (or one thread reads while another is initializing), ther
- **Suggested Fix:** These are typically set once during module initialization. Use `std::atomic<PyObject*>` or `std::once_flag`/`std::call_once` to ensure initialization is safe. Alternatively, rely on Python's own impor

### 5. Lazy-init static `PyCodeObject*` caches in `getCode<>()` templates
- **Severity:** High | **Confidence:** High
- **Location:** `torch/csrc/autograd/profiler_python.cpp:69-81, 84-96`
- **Report:** [autograd__part3](autograd__part3.md)
- **Description:** `getCode<CallType::PyModuleCall>()` and `getCode<CallType::PyOptimizerCall>()` each use a `static auto code = []() { ... }()` pattern that acquires the GIL inside the lambda. While C++ guarantees thread-safe initialization of function-local statics, the stored value is a bare `PyCodeObject*` (borrow
- **Suggested Fix:** `Py_INCREF(res)` before storing it in the static, so the `PyCodeObject*` is a strong reference that prevents the object from being collected.

### 6. Global `cpp_function_types_map` and `cpp_function_types_set` without synchronization
- **Severity:** High | **Confidence:** High
- **Location:** `torch/csrc/autograd/python_cpp_function.cpp:269-270, 300-306, 321-326, 328-339`
- **Report:** [autograd__part3](autograd__part3.md)
- **Description:** `cpp_function_types_map` and `cpp_function_types_set` are static globals that are written by `registerCppFunction` and read by `functionToPyObject` and `THPCppFunction_Check`. Under the GIL these were safe because all Python-facing calls were serialized. Under free-threading, concurrent calls to `fu
- **Suggested Fix:** Protect with a `std::mutex`, or use a concurrent data structure. If registration only happens during module init, consider a read-only pattern after initialization is complete (e.g., `folly::Synchroni

### 7. `functionToPyObject` TOCTOU race on `cdata->pyobj()`
- **Severity:** High | **Confidence:** High
- **Location:** `torch/csrc/autograd/python_cpp_function.cpp:296-319`
- **Report:** [autograd__part3](autograd__part3.md)
- **Description:** `functionToPyObject` first checks `cdata->pyobj()` is null (line 296), then if null creates a new Python wrapper and calls `cdata->set_pyobj(obj.release())` (line 315), and finally returns `cdata->pyobj()` (line 318). If the object already has a pyobj, it does `Py_INCREF(cdata->pyobj())` (line 297).
- **Suggested Fix:** Use a mutex or atomic compare-and-swap when setting `pyobj()`, or use `Py_BEGIN_CRITICAL_SECTION` on the node to serialize access.

### 8. Lazy-init pattern in `get_base_setup_context` with static `THPFunction_setup_context`
- **Severity:** High | **Confidence:** High
- **Location:** `torch/csrc/autograd/python_function.cpp:1307-1330`
- **Report:** [autograd__part4](autograd__part4.md)
- **Description:** `THPFunction_setup_context` is a file-static `PyObject*` that is lazily initialized in `get_base_setup_context()`. Under free-threading, multiple threads can race on the null check and the assignment, leading to duplicate initialization, leaked references, or use of a partially-written pointer. This
- **Suggested Fix:** Use `std::call_once` or an atomic pointer with acquire/release semantics to ensure the initialization runs exactly once. Alternatively, initialize this eagerly during module init.

### 9. Lazy-init caching pattern in `DEFINE_CACHING_PYTHON_IMPORT_GETTER` (pre-pybind 2.13)
- **Severity:** High | **Confidence:** High
- **Location:** `torch/csrc/autograd/python_variable.cpp:839-844`
- **Report:** [autograd__part5](autograd__part5.md)
- **Description:** When `IS_PYBIND_2_13_PLUS` is *not* defined, the `DEFINE_CACHING_PYTHON_IMPORT_GETTER` macro uses a plain `static py::handle` with no synchronization. If two threads call one of these getters concurrently (e.g., `get_dtensor_class_impl()`), they may both execute the initialization simultaneously, ca
- **Suggested Fix:** This is already fixed in the `IS_PYBIND_2_13_PLUS` path which uses `py::gil_safe_call_once_and_store`. Ensure that the pybind11 version used with free-threading builds is always 2.13+, or backport the

### 10. Global mutable `current_custom_allocator` shared_ptr without synchronization
- **Severity:** High | **Confidence:** High
- **Location:** `torch/csrc/cuda/CUDAPluggableAllocator.cpp:365, 369, 389-390, 394`
- **Report:** [cuda__part1](cuda__part1.md)
- **Description:** The namespace-scope `std::shared_ptr<CUDAAllocator> current_custom_allocator` is written by `changeCurrentAllocator` (line 390) and read by `getCurrentAllocator` (line 369) and `custom_raw_deleter` (line 394). `custom_raw_deleter` is the deleter registered with every allocation from the pluggable al
- **Suggested Fix:** Use `std::atomic<std::shared_ptr<...>>` (C++20) or protect all accesses with a mutex, or use `std::atomic_load`/`std::atomic_store` on the `shared_ptr`.

### 11. `PyGILState_Ensure` / `PyGILState_Release` in cuda mutex lock/unlock
- **Severity:** High | **Confidence:** High
- **Location:** `torch/csrc/cuda/Module.cpp:495, 513, 519`
- **Report:** [cuda__part1](cuda__part1.md)
- **Description:** `THCPModule_cudaLockMutex` calls `PyGILState_Ensure()` and stores the result in the file-static `cudaMutexGILState`, and `THCPModule_cudaUnlockMutex` calls `PyGILState_Release(cudaMutexGILState)`. The `PyGILState_Ensure`/`Release` API is deprecated and does not work correctly under free-threading (t
- **Suggested Fix:** Under free-threading, remove the `PyGILState_Ensure`/`PyGILState_Release` calls. The CUDA free mutex itself provides the necessary serialization for the CUDA operations; the GIL manipulation was only 

### 12. Static `InternedStringsTable` accessed without synchronization
- **Severity:** High | **Confidence:** High
- **Location:** `torch/csrc/python_dimname.cpp:24, 100-108`
- **Report:** [toplevel__part4](toplevel__part4.md)
- **Description:** `kPyInternedStringToDimname` is a file-static mutable `InternedStringsTable` instance containing a `ska::flat_hash_map`. `THPDimname_parse` (line 80) calls both `lookup` and `addMapping` on this table without any locking. Under the GIL, concurrent calls were serialized, but under free-threading mult
- **Suggested Fix:** Protect `kPyInternedStringToDimname` accesses with a `std::mutex`, or use a concurrent hash map. Alternatively, since the keys are interned Python strings (pointer-identity lookup), a `Py_BEGIN_CRITIC

### 13. Global mutable `is_initialized` / `is_in_bad_fork` / `at_fork_registered` arrays without synchronization
- **Severity:** High | **Confidence:** High
- **Location:** `torch/csrc/utils/device_lazy_init.cpp:15-17, 22-25, 62, 65-67, 69-75, 80`
- **Report:** [utils__part1](utils__part1.md)
- **Description:** The file-scope `std::array<bool, ...>` globals `is_initialized`, `is_in_bad_fork`, and `at_fork_registered` are read and written from multiple functions (`device_lazy_init`, `set_requires_device_init`, `is_device_in_bad_fork`, `set_device_in_bad_fork`, `register_fork_handler_for_device_init`). The c
- **Suggested Fix:** Replace each `bool` element with `std::atomic<bool>`, or protect the critical sections with a mutex. The `device_lazy_init` function already acquires the GIL (line 23); under free-threading, this `gil

### 14. `device_lazy_init` relies on `pybind11::gil_scoped_acquire` for mutual exclusion
- **Severity:** High | **Confidence:** High
- **Location:** `torch/csrc/utils/device_lazy_init.cpp:28-29`
- **Report:** [utils__part1](utils__part1.md)
- **Description:** `device_lazy_init` uses `pybind11::gil_scoped_acquire` and the comment explicitly states "Protected by the GIL." Under free-threading, acquiring the GIL is a no-op (or acquires a non-exclusive lock), so this no longer serializes access. The check-then-act pattern (`if (is_device_initialized(...)) re
- **Suggested Fix:** Introduce a per-device-type `std::mutex` (or a single mutex guarding the whole array) and hold it across the check-init-set sequence. Alternatively, use `c10::call_once` with per-device `c10::once_fla

### 15. Static global `PyObject*` for `disabled_torch_function` / `disabled_torch_dispatch`
- **Severity:** High | **Confidence:** High
- **Location:** `torch/csrc/utils/disable_torch_function.cpp:10-11, 18-24, 26-32`
- **Report:** [utils__part1](utils__part1.md)
- **Description:** `disabled_torch_function` and `disabled_torch_dispatch` are file-scope static `PyObject*` pointers that are read by `disabled_torch_function_impl()` / `disabled_torch_dispatch_impl()` and written by their `set_*` counterparts. Under the GIL these accesses were serialized. Without the GIL, concurrent
- **Suggested Fix:** Use `std::atomic<PyObject*>` for both globals, or protect accesses with a lightweight lock. Since these are essentially set once during module init and then only read, an atomic with `memory_order_acq

## Medium Severity Issues (60 total)

| # | File | Issue | Confidence |
|---|------|-------|------------|
| 1 | `?` | Unprotected Global Mutable State: `upgrader_registry` | Unknown |
| 2 | `?` | Borrowed References from Python Containers Without Critical Section | Unknown |
| 3 | `?` | Direct Python Object Mutation Without Critical Section | Unknown |
| 4 | `?` | Mutable Static `default_backend` Without Synchronization | Unknown |
| 5 | `CudaIPCTypes.cpp` | `GetNewRefCountedSentData` accesses shared state outside mutex | High |
| 6 | `DataLoader.cpp` | `PyTuple_GET_ITEM` returns borrowed references in `THPModule_setWorker | Medium |
| 7 | `DynamicTypes.cpp` | Lazy-init static `PyTypeObject*` cache in `getTypedStorageTypeObject` | High |
| 8 | `DynamicTypes.cpp` | Global `dtype_registry` and `layout_registry` arrays without synchroni | Medium |
| 9 | `Exceptions.h` | Global mutable exception `PyObject*` pointers in `Exceptions.h` | Medium |
| 10 | `Generator.cpp` | Global mutable `THPGeneratorClass` PyObject pointer | Medium |
| 11 | `Generator.cpp` | `THPGenerator_Wrap` TOCTOU race on pyobj check | Medium |
| 12 | `Generator.cpp` | `THPGenerator_pickleSetState` uses borrowed `PyTuple_GET_ITEM` referen | Medium |
| 13 | `Module.cpp` | `THPModule_initNames` uses unprotected static `std::vector<std::string | Medium |
| 14 | `Module.cpp` | `THPModule_addDocStr` uses unprotected static `std::vector<std::string | Medium |
| 15 | `Storage.cpp` | Global mutable `THPStorageClass` pointer set during module post-init | Medium |
| 16 | `Stream.cpp` | Global mutable `THPStreamClass` pointer set during module init | Medium |
| 17 | `Stream.cpp` | `THPStream.context` list accessed without critical section | Medium |
| 18 | `autograd/forward_grad.h` | `ForwardGrad::empty()` reads `content_` without holding the mutex | Medium |
| 19 | `autograd/function.h` | `Node::tensor_post_acc_grad_hooks()` returns reference to static local | High |
| 20 | `autograd/generated/Functions.cpp` | Non-atomic `static bool called` race in every `apply_with_saved` metho | High |
| 21 | `autograd/jit_decomp_interface.cpp` | Global mutable `JitDecompInterface*` pointer without synchronization | Medium |
| 22 | `autograd/profiler_kineto.cpp` | `profiler_state_info_ptr` shared_ptr global without synchronization | Medium |
| 23 | `autograd/profiler_python.cpp` | Static `py_gc_callback` global PyObject pointer without synchronizatio | Medium |
| 24 | `autograd/profiler_python.cpp` | `PyGILState_Ensure`/`Release` usage in profiler_python.cpp | High |
| 25 | `autograd/profiler_python.cpp` | Borrowed reference from `PyDict_GetItemString` in `recordPyCall` | Medium |
| 26 | `autograd/python_engine.cpp` | Static `_reinitialize_engine` flag without atomic access | Medium |
| 27 | `autograd/python_function.cpp` | Global mutable `THPFunctionClass` and `THPGradientEdgeClass` without s | Medium |
| 28 | `autograd/python_function.cpp` | `PyDict_GetItemString` returns borrowed reference in `_trace_post_reco | Medium |
| 29 | `autograd/python_function.cpp` | `PyNode::compiled_args` uses `static PyObject* method_name` lazy init | High |
| 30 | `autograd/python_nested_functions_manual.cpp` | `static PythonArgParser` in `THPVariable_nested_tensor` | Medium |
| 31 | `autograd/python_torch_functions_manual.cpp` | Global mutable `THPVariableFunctionsModule` PyObject pointer | Medium |
| 32 | `autograd/python_variable.cpp` | Global mutable `THPVariableClass` and `ParameterClass` PyObject pointe | Medium |
| 33 | `autograd/python_variable.cpp` | Global mutable `device_to_py_class_` array | Medium |
| 34 | `autograd/python_variable.cpp` | `PyDict_GetItemWithError` returns borrowed reference | Medium |
| 35 | `backend_detail.cpp` | backendPreprocessFunctions() -- Unprotected static mutable map (Medium | Unknown |
| 36 | `cuda/Event.cpp` | Static `PythonArgParser` in `THCPEvent_from_ipc_handle` is not thread- | High |
| 37 | `cuda/Module.cpp` | `THCPModule_attachOutOfMemoryObserver` captures PyObject without threa | Medium |
| 38 | `cuda/Module.cpp` | Borrowed references from `PyTuple_GetItem` stored in `THPObjectPtr` | High |
| 39 | `cuda/nccl.cpp` | Static `_communicators` map accessed under CudaFreeMutex assumption | Medium |
| 40 | `function_impl.cpp` | GraphFunction::ensure_defined() -- Race on function_creator_ (Medium) | Unknown |
| 41 | `function_impl.cpp` | GraphFunction::getSchema() -- Lazy init race on schema_ (Medium) | Unknown |
| 42 | `jit/mobile/code.h` | `Code::initialized` flag (code.h, function.cpp) | Unknown |
| 43 | `jit/mobile/function.cpp` | Static mutable upgrader function holder (function.cpp) | Unknown |
| 44 | `jit/mobile/nnc/context.cpp` | `Function::init_execution_state()` lazy initialization (context.cpp) | Unknown |
| 45 | `jit/mobile/nnc/context.cpp` | `Function::run()` mutates `execution_state_->arguments_` (context.cpp) | Unknown |
| 46 | `jit/mobile/prim_ops_registery.cpp` | Static mutable prim ops function table (prim_ops_registery.cpp) | Unknown |
| 47 | `jit/mobile/upgrader_mobile.cpp` | Static upgrader bytecode list in upgrader_mobile.cpp | Unknown |
| 48 | `jit_log.cpp` | JitLoggingConfig singleton -- Unsynchronized mutable state (Medium) | Unknown |
| 49 | `jit_opt_limit.cpp` | passes_to_current_counter() -- Unprotected mutable static map (Medium) | Unknown |
| 50 | `nnapi_backend_lib.cpp` | NnapiBackend::execute() -- Lazy init of comp_ and out_templates_ (Medi | Unknown |
| 51 | `python_dimname.cpp` | Borrowed references from Python containers in `THPUtils_checkDimnameLi | Medium |
| 52 | `utils.cpp` | Borrowed references from Python containers in `THPUtils_unpackLongs` a | Medium |
| 53 | `utils.cpp` | Borrowed references in pybind11 `type_caster` load methods | Medium |
| 54 | `utils.cpp` | `PyGILState_Ensure`/`Release` in `torch::gdb::tensor_repr` | Medium |
| 55 | `utils/device_lazy_init.cpp` | `is_device_initialized` acquires GIL but the read is still racy | High |
| 56 | `utils/disable_torch_function.cpp` | `disabled_torch_function` and `disabled_torch_dispatch` global PyObjec | High |
| 57 | `utils/disable_torch_function.cpp` | `has_torch_function_attr` uses borrowed reference from `PyObject_FastG | Medium |
| 58 | `utils/disable_torch_function.cpp` | `sequence_has_torch_function` uses `PySequence_Fast_GET_ITEM` (borrowe | Medium |
| 59 | `utils/nested.cpp` | `nested_tensor_ctor` uses `PyList_Size` and `PyList_GetItemRef` withou | Medium |
| 60 | `utils/tensor_new.cpp:1385` | `static std::string sig` in template function | Unknown |

## Low Severity Issues (71 total)

These are typically write-once-during-init patterns, unlikely races, or non-atomic flags that should be `std::atomic` but have low practical impact.

## Systemic Patterns Requiring Attention

### 1. Static `PythonArgParser` instances
Many Python binding functions use `static PythonArgParser` which mutates internal state during parsing. Under free-threading, concurrent calls from multiple Python threads race on this shared mutable state. This is pervasive across `torch/csrc/utils/`, `autograd/`, `jit/python/`, and `cuda/`.

### 2. Write-once-then-read global `PyObject*` pointers
Many global `PyObject*` pointers (e.g., `THPVariableClass`, `THPFunctionClass`, exception types) are set once during module init and then read-only. These are technically races under free-threading but are low risk since Python's import lock serializes module init. Making them `std::atomic<PyObject*>` would be defensive.

### 3. `PyDict_GetItem` / `PyList_GET_ITEM` borrowed references
These return borrowed references that can dangle if another thread modifies the container. Should be replaced with `PyDict_GetItemRef` (3.13+) or wrapped in `Py_BEGIN_CRITICAL_SECTION`.

### 4. Unprotected static caches and registries
`static std::unordered_map`, `static std::unordered_set`, `static std::vector` used as caches in Python-facing functions. These relied on GIL for serialization. Need `std::mutex` or concurrent data structures.

### 5. `PyGILState_Ensure`/`Release` incompatibility
Several call sites use `PyGILState_Ensure`/`Release` which is incompatible with free-threading. These need to be replaced with `Py_BEGIN_CRITICAL_SECTION` or removed if the surrounding mutex already provides sufficient serialization.

### 6. Non-atomic C++ global flags
Many `static bool` / `static int` flags are read/written from Python-facing functions without `std::atomic`. While the GIL used to serialize these, free-threading exposes them as data races.

## Highest-Risk Files

- `torch/csrc/autograd/profiler_python.cpp` — 4 issues
- `torch/csrc/autograd/python_function.cpp` — 4 issues
- `torch/csrc/autograd/python_variable.cpp` — 4 issues
- `torch/csrc/utils/disable_torch_function.cpp` — 4 issues
- `torch/csrc/Module.cpp` — 3 issues
- `torch/csrc/cuda/Module.cpp` — 3 issues
- `torch/csrc/utils/device_lazy_init.cpp` — 3 issues
- `torch/csrc/Generator.cpp` — 3 issues
- `torch/csrc/utils.cpp` — 3 issues
- `torch/csrc/DataLoader.cpp` — 2 issues
- `torch/csrc/autograd/python_cpp_function.cpp` — 2 issues
- `torch/csrc/python_dimname.cpp` — 2 issues
- `torch/csrc/DynamicTypes.cpp` — 2 issues
- `torch/csrc/Stream.cpp` — 2 issues
- `torch/csrc/function_impl.cpp` — 2 issues
- `torch/csrc/jit/mobile/nnc/context.cpp` — 2 issues
- `torch/csrc/autograd/init.cpp` — 1 issues
- `torch/csrc/cuda/CUDAPluggableAllocator.cpp` — 1 issues
- `torch/csrc/CudaIPCTypes.cpp` — 1 issues
- `torch/csrc/Exceptions.h` — 1 issues

## Areas With No Issues

55 report groups had no free-threading issues. These include:
- `api/` (C++ frontend) — 35 groups, all clean
- `autograd/functions/`, `autograd/utils/` — already mutex-protected
- `distributed/autograd/` — already designed for multi-threaded RPC
- `cpu/`, `functionalization/`, `functorch/`, `instruction_counter/`, `monitor/`, `mps/`, `mtia/`, `multiprocessing/`, `onnx/`, `stable/`