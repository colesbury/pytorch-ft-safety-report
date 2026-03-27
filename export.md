# Free-Threading Safety Audit: export

**Overall Risk: Medium**

## Files Reviewed
- `export/example_upgraders.cpp`
- `export/example_upgraders.h`
- `export/pt2_archive_constants.h`
- `export/pybind.cpp`
- `export/pybind.h`
- `export/upgrader.cpp`
- `export/upgrader.h`

## Findings

### 1. Unprotected Global Mutable State: `upgrader_registry`
**Severity: Medium**
**Location:** `export/upgrader.cpp`, line 15

```cpp
static std::map<int, std::multiset<Upgrader>> upgrader_registry;
```

The `upgrader_registry` is a static mutable map that is read and written by multiple functions (`registerUpgrader`, `deregisterUpgrader`, `getUpgrader`, `upgrade`) without any synchronization. Under free-threading, concurrent calls to these functions from Python (via pybind11 bindings in `pybind.cpp`) would cause data races.

Affected functions:
- `registerUpgrader()` (lines 64-113) -- writes to registry
- `deregisterUpgrader()` (lines 115-159) -- writes to registry
- `getUpgrader()` (lines 17-24) -- reads from registry
- `upgrade()` (lines 178-233) -- reads from registry

### 2. Unprotected Static Boolean Guard: `test_upgraders_registered`
**Severity: Low**
**Location:** `export/example_upgraders.cpp`, line 8

```cpp
static bool test_upgraders_registered = false;
```

The `test_upgraders_registered` flag is checked and set without synchronization in `registerExampleUpgraders()` and `deregisterExampleUpgraders()`. This is a classic TOCTOU race. However, this is primarily for testing, so the practical risk is low.

### 3. Python Module Import During Deserialization
**Severity: Low**
**Location:** `export/pybind.cpp`, lines 25-27

```cpp
py::module_ schema_module =
    py::module_::import("torch._export.serde.schema");
py::tuple schema_version_tuple = schema_module.attr("SCHEMA_VERSION");
```

The `deserialize_exported_program` lambda imports a Python module and accesses its attributes. Under free-threading, `PyImport_ImportModule` should be thread-safe (it uses internal locks), but the subsequent attribute access on `schema_module` uses a borrowed reference without critical section protection. This is low risk as pybind11 manages the GIL currently.

## Summary

The main concern is the `upgrader_registry` global mutable map in `upgrader.cpp`, which is accessed by both registration and upgrade functions without any mutex or other synchronization. If `register_example_upgraders`, `deregister_example_upgraders`, or `upgrade` are called concurrently from multiple threads, data races will occur. A mutex should be added to protect the registry.
