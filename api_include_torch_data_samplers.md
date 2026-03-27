# Free-Threading Safety Report: api_include_torch_data_samplers

**Overall Risk: None**

## Files Analyzed
- `api/include/torch/data/samplers/base.h`
- `api/include/torch/data/samplers/custom_batch_request.h`
- `api/include/torch/data/samplers/distributed.h`
- `api/include/torch/data/samplers/random.h`
- `api/include/torch/data/samplers/sequential.h`
- `api/include/torch/data/samplers/serialize.h`
- `api/include/torch/data/samplers/stream.h`

## Analysis

No free-threading issues found.

These are pure C++ sampler class declarations and definitions for the libtorch data API. All state is instance-level. No Python C API usage. No static/global mutable state.
