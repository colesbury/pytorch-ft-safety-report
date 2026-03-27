# Free-Threading Safety Audit: utils (part 5)

## Files Reviewed
- `torch/csrc/utils/tensor_numpy.cpp`
- `torch/csrc/utils/tensor_numpy.h`
- `torch/csrc/utils/tensor_qschemes.cpp`
- `torch/csrc/utils/tensor_qschemes.h`
- `torch/csrc/utils/tensor_types.cpp`
- `torch/csrc/utils/tensor_types.h`
- `torch/csrc/utils/throughput_benchmark.cpp`
- `torch/csrc/utils/throughput_benchmark.h`
- `torch/csrc/utils/throughput_benchmark-inl.h`
- `torch/csrc/utils/torch_dispatch_mode.h`
- `torch/csrc/utils/variadic.cpp`
- `torch/csrc/utils/variadic.h`
- `torch/csrc/utils/verbose.cpp`
- `torch/csrc/utils/verbose.h`

## Overall Risk: Medium

## Issues

### Issue 1 — Static mutable `PyObject*` array without synchronization (tensor_qschemes.cpp)
**Severity:** Medium
**Location:** `torch/csrc/utils/tensor_qschemes.cpp`, lines 13, 15-31, 33-39

The file declares a static global array of `PyObject*`:
```cpp
static std::array<PyObject*, at::COMPILE_TIME_NUM_QSCHEMES> thp_qscheme_array;
```

`initializeQSchemes()` writes to this array and `getTHPQScheme()` reads from it. Under the GIL, the init-then-read pattern was safe because module initialization is serialized. Under free-threading, if two threads import the module concurrently, one could call `getTHPQScheme()` while another is still inside `initializeQSchemes()`, reading a partially-initialized array.

**Recommendation:** If `initializeQSchemes` is only called from module init (`Py_mod_exec`), CPython 3.14 serializes per-module init, so this is likely safe in practice. However, the global array itself is shared across interpreters. Consider using `std::call_once` or making the init check explicit if this can be called outside module init.

### Issue 2 — Static local lazy-init maps with data races (tensor_types.cpp)
**Severity:** High
**Location:** `torch/csrc/utils/tensor_types.cpp`, lines 80-142

`options_from_string()` uses multiple static local lazy-init patterns:
```cpp
static std::unordered_map<std::string, at::DeprecatedTypeProperties*> cpu_map;
...
static bool cpu_once [[maybe_unused]] = []() {
    for (auto type : autograd::VariableType::allCPUTypes()) {
        cpu_map.emplace(type_to_string(*type), type);
    }
    return true;
}();
```

There are four such maps (`cpu_map`, `cuda_map`, `xpu_map`, `privateUser1_map`) and four corresponding `static bool` guard variables. The `static bool` initialization is thread-safe (C++ guarantees that), but the maps themselves are declared as separate static locals. The `static bool` lambda captures the map by reference and populates it. After the `static bool` is initialized, the map is read via `map->find(str)`.

The critical issue: the `static bool` init is thread-safe, but there is no happens-before relationship between the `static bool` initialization completing and subsequent reads from the separately-declared static map. The map is a different static variable from the `static bool`, so the C++ static-init guarantee does not cover it. Two threads could enter `options_from_string` concurrently; one triggers the `static bool` init (which populates the map), while the other reads the map before it is fully populated.

**Recommendation:** Combine each map and its initialization into a single `static` local that is the map itself, returned from the lambda:
```cpp
static auto& cpu_map = *new std::unordered_map<...>([]() {
    std::unordered_map<...> m;
    for (auto type : autograd::VariableType::allCPUTypes())
        m.emplace(type_to_string(*type), type);
    return m;
}());
```
This way the C++ static-init thread-safety guarantee covers the fully-populated map.

### Issue 3 — Static local strings with potential data race (tensor_types.cpp)
**Severity:** Low
**Location:** `torch/csrc/utils/tensor_types.cpp`, lines 17-22

```cpp
static const char* parse_privateuseone_backend(bool is_sparse = false) {
  static std::string backend_name = "torch." + get_privateuse1_backend();
  static std::string sparse_backend_name = backend_name + ".sparse";
  return ...;
}
```

These are `static` locals so their initialization is thread-safe by C++ guarantees. The returned `c_str()` pointer is stable as long as the string is not modified, and since these are `const` after init, this is safe.

However, `options_from_string` also uses these indirectly:
```cpp
static std::string privateUser_prefix(
    std::string(parse_privateuseone_backend()) + ".");
```
This is a `static` local `std::string`, which is also safe due to C++ static init guarantees.

**No fix needed** -- this is safe.

### Issue 4 — `is_numpy_available()` lazy init is safe but `validate_numpy_for_dlpack_deleter_bug` has a non-atomic write (tensor_numpy.cpp)
**Severity:** Low
**Location:** `torch/csrc/utils/tensor_numpy.cpp`, lines 529, 551-583

```cpp
static bool numpy_with_dlpack_deleter_bug_installed = false;
```

This global `bool` is written in `validate_numpy_for_dlpack_deleter_bug()` and read in `is_numpy_dlpack_deleter_bugged()`. The validate function contains its own guard:
```cpp
static bool validated = false;
TORCH_INTERNAL_ASSERT(validated == false);
validated = true;
```

The `validated` guard is itself a non-atomic static local written without synchronization (not a `static` local with an initializer, but a `static` local that is mutated via assignment). If called from module init, CPython serializes this, so it is likely safe in practice. `numpy_with_dlpack_deleter_bug_installed` is written once during init and then only read, so the init-then-read pattern is safe if module init provides the happens-before.

**Recommendation:** Mark `numpy_with_dlpack_deleter_bug_installed` as `std::atomic<bool>` for correctness if there's any chance it could be read before module init completes. The `validated` guard variable could similarly use `std::call_once`.

### Issue 5 — `pybind11::gil_scoped_acquire` usage in throughput benchmark (throughput_benchmark.cpp)
**Severity:** Low
**Location:** `torch/csrc/utils/throughput_benchmark.cpp`, lines 89, 98, 126

The `ModuleBenchmark::runOnce` and `cloneInput<ModuleInput>` use `pybind11::gil_scoped_acquire`. Under free-threading, the GIL no longer exists so these calls become no-ops. The benchmark intentionally uses GIL-based serialization when running `nn.Module` forward passes from multiple threads. Under free-threading, this serialization disappears, potentially exposing races in user Python code -- but that is a user-code concern, not a PyTorch internals issue.

**Recommendation:** No immediate fix needed in PyTorch. The benchmark already warns that `nn.Module` benchmarking can be slow due to the GIL; under free-threading this warning should be updated.

## Files With No Issues
- `torch/csrc/utils/tensor_numpy.h` -- pure declarations
- `torch/csrc/utils/tensor_qschemes.h` -- pure declarations
- `torch/csrc/utils/tensor_types.h` -- pure declarations
- `torch/csrc/utils/throughput_benchmark.h` -- class declarations, no global state
- `torch/csrc/utils/throughput_benchmark-inl.h` -- template method uses proper mutex/condvar synchronization for its own threading
- `torch/csrc/utils/torch_dispatch_mode.h` -- operates on TLS state only, no global mutable state
- `torch/csrc/utils/variadic.cpp` -- empty (just includes header)
- `torch/csrc/utils/variadic.h` -- pure templates, no state
- `torch/csrc/utils/verbose.cpp` -- module submodule init via pybind, no global state
- `torch/csrc/utils/verbose.h` -- pure declaration

## Summary

The most significant issue is in `tensor_types.cpp` where `options_from_string()` uses a pattern of separate static maps populated via separate `static bool` guard lambdas. The maps are not covered by the C++ static initialization thread-safety guarantee since they are distinct static variables from the guards. Under free-threading, concurrent calls could read partially-populated maps. The fix is to make each map a single `static` local initialized by a lambda so the C++ guarantee covers both creation and population.

The `thp_qscheme_array` in `tensor_qschemes.cpp` is a global mutable array populated during module init and read afterwards. This is safe if module init provides a happens-before guarantee, which CPython 3.14 does for `Py_mod_exec`. Still, making this more robust (e.g., `std::call_once`) would be defensive.
