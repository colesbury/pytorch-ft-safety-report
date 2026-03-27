# Free-Threading Safety Audit: api_src_data_samplers

**Overall Risk: None**

## Files Reviewed
- `api/src/data/samplers/distributed.cpp`
- `api/src/data/samplers/random.cpp`
- `api/src/data/samplers/sequential.cpp`
- `api/src/data/samplers/stream.cpp`

## Summary

No free-threading issues found. These files are part of the pure C++ frontend (libtorch) and do not interact with the Python C API. All state is instance-level (member variables like `sample_index_`, `indices_`, `epoch_size_`). No static or global mutable state. No Python objects.
