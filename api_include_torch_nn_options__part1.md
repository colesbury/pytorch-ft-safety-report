# Free-Threading Safety Report: api_include_torch_nn_options__part1

**Overall Risk: None**

## Files Analyzed
- `api/include/torch/nn/options/activation.h`
- `api/include/torch/nn/options/adaptive.h`
- `api/include/torch/nn/options/batchnorm.h`
- `api/include/torch/nn/options/conv.h`
- `api/include/torch/nn/options/distance.h`
- `api/include/torch/nn/options/dropout.h`
- `api/include/torch/nn/options/embedding.h`
- `api/include/torch/nn/options/fold.h`
- `api/include/torch/nn/options/instancenorm.h`
- `api/include/torch/nn/options/linear.h`
- `api/include/torch/nn/options/loss.h`
- `api/include/torch/nn/options/normalization.h`
- `api/include/torch/nn/options/padding.h`
- `api/include/torch/nn/options/pixelshuffle.h`
- `api/include/torch/nn/options/pooling.h`

## Analysis

No free-threading issues found.

These are pure C++ options structs generated via `TORCH_ARG` macros. Each struct holds configuration values (kernel sizes, padding modes, learning rates, etc.) as instance-level members. The `TORCH_ARG` macro generates getter/setter methods. No Python C API usage. No static/global mutable state.

Note: `activation.h` has `TORCH_ARG(Tensor, static_k)` and `TORCH_ARG(Tensor, static_v)` members in `MultiheadAttentionForwardFuncOptions`. Despite the name `static_k`/`static_v`, these are instance-level tensor members (the "static" refers to static key/value in the attention mechanism, not C++ storage class).
