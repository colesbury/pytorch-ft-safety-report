# `load_json_file` static JSON object

- **Status:** Open
- **Severity:** SEVERE
- **Component:** aoti_package/model_package_loader.cpp

- **Shared state:** `static nlohmann::json json_obj` in `load_json_file()`
  (`model_package_loader.cpp:159`)
- **Writer:** Every call to `load_json_file()` overwrites this static object.
- **Reader:** The caller holds a reference to the returned object.
- **Race scenario:** Thread A calls `load_json_file("model_a.json")`, gets a
  reference to the static. Thread B calls `load_json_file("model_b.json")`,
  overwrites the static. Thread A's reference now points to model_b's data
  (or worse, a partially-overwritten JSON tree). `nlohmann::json` uses
  `std::map`/`std::vector` internally — concurrent read + overwrite is UB.
- **Consequence:** Crash or silent data corruption (wrong model config loaded).
  This is also a latent single-threaded bug since the returned reference can
  be invalidated by a subsequent call.
- **Suggested fix:** Make the variable local and return by value.
