# Jit Free-Threading Issues

Issues from broad audit of `torch/csrc/jit/`. These have NOT been deep-audited — some may be false positives (config flags, write-once globals).

## Medium Severity

| File | Issue |
|------|-------|
| `jit_log.cpp` | JitLoggingConfig singleton -- Unsynchronized mutable state (Medium) |
| `jit_opt_limit.cpp` | passes_to_current_counter() -- Unprotected mutable static map (Medium) |
| `function_impl.cpp` | GraphFunction::ensure_defined() -- Race on function_creator_ (Medium) |
| `function_impl.cpp` | GraphFunction::getSchema() -- Lazy init race on schema_ (Medium) |
| `backend_detail.cpp` | backendPreprocessFunctions() -- Unprotected static mutable map (Medium |
| `nnapi_backend_lib.cpp` | NnapiBackend::execute() -- Lazy init of comp_ and out_templates_ (Medi |
| `jit/mobile/nnc/context.cpp` | `Function::init_execution_state()` lazy initialization (context.cpp) |
| `jit/mobile/nnc/context.cpp` | `Function::run()` mutates `execution_state_->arguments_` (context.cpp) |

*8 issues from broad audit. Deep audit needed to confirm.*
