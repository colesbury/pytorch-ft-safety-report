# Free-Threading Safety Audit: api_src_serialize

**Overall Risk: None**

## Files Reviewed
- `api/src/serialize/input-archive.cpp`
- `api/src/serialize/output-archive.cpp`

## Summary

No free-threading issues found. These files are part of the pure C++ frontend (libtorch) and do not interact with the Python C API.

- **input-archive.cpp**: `InputArchive` wraps a `jit::Module` and provides `read`/`try_read`/`load_from` methods. All state is instance-level (`module_`, `hierarchy_prefix_`). The `load_from` overloads create local adapter objects. No static or global mutable state. No Python objects.
- **output-archive.cpp**: `OutputArchive` wraps a `jit::Module` and provides `write`/`save_to` methods. All state is instance-level (`cu_`, `module_`). No static or global mutable state. No Python objects.
