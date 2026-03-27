# Free-Threading Safety Audit: inductor_aoti_package

## Overall Risk: LOW

## Files Reviewed
- `inductor/aoti_package/model_package_loader.cpp`
- `inductor/aoti_package/model_package_loader.h`
- `inductor/aoti_package/pybind.cpp`
- `inductor/aoti_package/pybind.h`

## Detailed Analysis

### model_package_loader.cpp

**Issue: `static nlohmann::json json_obj` in `load_json_file`** (line 159)

```cpp
const nlohmann::json& load_json_file(const std::string& json_path) {
  ...
  static nlohmann::json json_obj;
  json_file >> json_obj;
  return json_obj;
}
```

This function uses a `static` local variable that is mutated on every call (overwritten by `json_file >> json_obj`). If two threads call `load_json_file` concurrently, they would race on writing to and reading from the same static object. However, this function is only called from the `AOTIModelPackageLoader` constructor and `load_metadata_from_package`, both of which operate on constructor-local data. In practice, concurrent construction of package loaders that call `load_json_file` simultaneously could trigger a data race.

That said, this is a pre-existing bug (it was already racy even with the GIL because the GIL doesn't protect C++ code executing without the GIL held). The `static` here appears to be an optimization mistake -- it should be a local variable. This is a latent bug rather than a free-threading regression.

**getAOTIModelRunnerRegistry**: Read at runtime to look up the runner factory. Populated at static-init time. Safe for read-only access.

**No Python C API in the core loader**: The C++ `AOTIModelPackageLoader` class does not interact with the Python C API directly. File I/O and zip extraction are pure C++ operations.

### pybind.cpp

`initAOTIPackageBindings` registers pybind11 bindings. Called once during module initialization. The `AOTIModelPackageLoaderPybind::boxed_run` method calls `inputs.attr("clear")()` and `THPVariable_Wrap`, both of which operate on Python objects. These calls happen through pybind11 which manages the GIL appropriately.

No issues specific to free-threading.

## Issues

| # | Severity | Description | Location |
|---|----------|-------------|----------|
| 1 | Low | `static nlohmann::json json_obj` in `load_json_file` is mutated on every call, creating a data race if called concurrently. Pre-existing bug, not a free-threading regression. | `model_package_loader.cpp:159` |

## Summary

One pre-existing bug with a static mutable JSON object that should be a local variable. No new free-threading regressions. The pybind layer correctly interacts with Python through pybind11's GIL management.
