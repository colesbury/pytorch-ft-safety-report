# Free-Threading Safety Report: api_include_torch_serialize

**Overall Risk: None**

## Files Analyzed
- `api/include/torch/serialize/archive.h`
- `api/include/torch/serialize/input-archive.h`
- `api/include/torch/serialize/output-archive.h`
- `api/include/torch/serialize/tensor.h`

## Analysis

No free-threading issues found.

- `archive.h`: Aggregator header including input and output archive headers.
- `input-archive.h`: `InputArchive` class for deserializing models. All state is instance-level (stored `c10::IValue`, file paths, etc.).
- `output-archive.h`: `OutputArchive` class for serializing models. All state is instance-level.
- `tensor.h`: `save` and `load` free functions for individual tensors. Stateless functions operating on arguments.

No Python C API usage. No static/global mutable state.
