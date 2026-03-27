# Free-Threading Safety Report: api_include_torch_nn_modules__part1

**Overall Risk: None**

## Files Analyzed
- `api/include/torch/nn/modules/_functions.h`
- `api/include/torch/nn/modules/activation.h`
- `api/include/torch/nn/modules/adaptive.h`
- `api/include/torch/nn/modules/batchnorm.h`
- `api/include/torch/nn/modules/common.h`
- `api/include/torch/nn/modules/conv.h`
- `api/include/torch/nn/modules/distance.h`
- `api/include/torch/nn/modules/dropout.h`
- `api/include/torch/nn/modules/embedding.h`
- `api/include/torch/nn/modules/fold.h`
- `api/include/torch/nn/modules/instancenorm.h`
- `api/include/torch/nn/modules/linear.h`
- `api/include/torch/nn/modules/loss.h`
- `api/include/torch/nn/modules/normalization.h`
- `api/include/torch/nn/modules/padding.h`

## Analysis

No free-threading issues found.

These are C++ module class declarations for libtorch's `nn::` namespace. Each file declares module structs (e.g., `LinearImpl`, `Conv2dImpl`) that inherit from `nn::Module` or `nn::Cloneable`. All state is instance-level (weight tensors, bias tensors, options structs).

- `_functions.h`: Declares `CrossMapLRN2d` with `static` `forward`/`backward` methods. These are stateless static member functions (autograd function convention), not static mutable state.
- `embedding.h`: Has `static Embedding from_pretrained(...)` factory methods. Again, stateless static member functions.

No Python C API usage. No static/global mutable state.
