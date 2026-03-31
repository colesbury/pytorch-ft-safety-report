# `options_from_string` lazy-init maps with broken guard pattern

- **Status:** Open
- **Severity:** SEVERE
- **Component:** tensor_types
- **Source reports:** [utils__part5.md](../utils__part5.md)

- **Shared state:** Four file-scope static `std::unordered_map` instances
  (`cpu_map`, `cuda_map`, `xpu_map`, `privateUser1_map`) in
  `torch/csrc/utils/tensor_types.cpp` (lines 84-89), populated via
  separate `static bool` guard lambdas (lines 104-141).
- **Writer(s):**
  - The `static bool cpu_once = [&]() { ... cpu_map.emplace(...); ... }()`
    lambda (and similar for cuda, xpu, privateUser1). These execute on
    first call for each branch and populate the corresponding map.
- **Reader(s):**
  - `options_from_string()` reads `map->find(str)` (line 144) immediately
    after the guard variable is initialized, on every call.
- **Race scenario:** The `static bool` guard and the `static unordered_map`
  are separate static local variables. C++ guarantees thread-safe
  initialization of each individual `static` local, but does NOT
  guarantee a happens-before relationship between the initialization of
  one static local and reads of a different static local.
  Thread A enters `options_from_string("torch.cuda.FloatTensor")` and
  triggers the initialization of `cuda_once`, which begins populating
  `cuda_map` via `emplace`. Thread B enters with the same prefix;
  the C++ runtime sees `cuda_once` is being initialized and blocks on
  it. But a third Thread C could enter with a different string that
  also hits the cuda branch -- after `cuda_once` initialization
  completes, it proceeds to `map->find(str)`. However, `cuda_map`
  is a different static variable whose initialization (to empty)
  happened separately. The population via `emplace` during the
  `cuda_once` lambda is not covered by `cuda_map`'s static-init
  synchronization (which already completed with the empty map).
  In practice the `static bool` init does provide a happens-before for
  threads that blocked on it, but any thread that already passed the
  `static bool` check (because it completed) could observe the map in
  a partially-populated state if the compiler/CPU reorders the map
  writes relative to the `static bool` completion flag. This is a
  subtle memory ordering issue that may manifest as missing entries.
- **Suggested fix:** Combine each map and its population into a single
  static local:
  ```cpp
  static auto& cuda_map = *new std::unordered_map<...>([]() {
      std::unordered_map<...> m;
      for (auto type : autograd::VariableType::allCUDATypes())
          m.emplace(type_to_string(*type), type);
      return m;
  }());
  ```
  This way the C++ static-init thread-safety guarantee covers both
  creation and full population of the map.
