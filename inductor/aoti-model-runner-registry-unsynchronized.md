# `getAOTIModelRunnerRegistry()` unsynchronized global `unordered_map`

- **Status:** Open
- **Severity:** SEVERE
- **Component:** aoti_runner/model_container_runner.cpp

- **Shared state:** Static `std::unordered_map<std::string, CreateAOTIModelRunnerFunc>`
  returned by `getAOTIModelRunnerRegistry()` (`model_container_runner.cpp:388`).
- **Writer:** `RegisterAOTIModelRunner` constructor writes to the map.
  The header says "It is not thread-safe. Because it is expected to be called
  during the initialization of the program."
- **Reader:** `AOTIPythonKernelHolder` constructor and `load_aoti_model_runner`
  call `find()`/`operator[]` on the map.
- **Race scenario:** Under free-threading, a lazy `import` on a background
  thread triggers a backend module's static initializer calling
  `RegisterAOTIModelRunner`, writing to the map. Main thread simultaneously
  reads the map to construct a kernel holder. Concurrent read + write on
  `unordered_map` is UB.
- **Tier:** 1 (lazy imports from data loader threads).
- **Suggested fix:** Wrap in `c10::Synchronized<>` or use `std::shared_mutex`.
