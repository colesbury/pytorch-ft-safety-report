# Free-Threading Safety Audit: jit/api

**Overall Risk: Medium**

## Files Reviewed
- `torch/csrc/jit/api/compilation_unit.h`
- `torch/csrc/jit/api/function_impl.cpp`
- `torch/csrc/jit/api/function_impl.h`
- `torch/csrc/jit/api/method.h`
- `torch/csrc/jit/api/module.cpp`
- `torch/csrc/jit/api/module.h`
- `torch/csrc/jit/api/module_save.cpp`
- `torch/csrc/jit/api/object.cpp`
- `torch/csrc/jit/api/object.h`

## Findings

### 1. GraphFunction::ensure_defined() -- Race on function_creator_ (Medium)
**File:** `function_impl.cpp`, lines 79-87
```cpp
void GraphFunction::ensure_defined() {
  if (function_creator_) {
    auto creator = function_creator_;
    function_creator_ = placeholderCreator;
    creator(*this);
    function_creator_ = nullptr;
  }
  check_single_output();
}
```
This is a classic check-then-act race. `function_creator_` is a `std::function` field with no synchronization. If two threads call `ensure_defined()` concurrently, both could read a truthy `function_creator_`, both could invoke the creator, and the writes to `function_creator_` could race with reads. Note that `get_executor()` calls `ensure_defined()` **before** acquiring `compile_mutex`, so the mutex does not protect this code path.

**Severity: Medium** -- Could cause double-initialization or use-after-move. However, this is JIT/TorchScript infrastructure and concurrent compilation of the same GraphFunction from multiple threads is unusual in practice.

**Recommendation:** Move the `ensure_defined()` call inside the `compile_mutex` lock in `get_executor()`, or add a separate mutex/`std::call_once` guard around the creator invocation.

### 2. GraphFunction::getSchema() -- Lazy init race on schema_ (Medium)
**File:** `function_impl.cpp`, lines 89-93
```cpp
const c10::FunctionSchema& GraphFunction::getSchema() const {
  if (schema_ == nullptr) {
    schema_ = std::make_unique<c10::FunctionSchema>(defaultSchemaFor(*this));
  }
  return *schema_;
}
```
`schema_` is a `mutable std::unique_ptr<FunctionSchema>` with no synchronization. Two threads calling `getSchema()` concurrently can both see `nullptr`, both allocate, and race on the assignment. One thread could also dereference `*schema_` while another is writing to it.

**Severity: Medium** -- Data race on mutable member. Same mitigation comment as above: JIT functions are not typically accessed from multiple threads simultaneously.

**Recommendation:** Protect with `compile_mutex` or use `std::call_once`.

### 3. thread_local inline_everything -- Safe
**File:** `module.cpp`, line 144
```cpp
static thread_local bool inline_everything = false;
```
Thread-local storage is inherently per-thread. No free-threading concern.

**Severity:** None

### 4. CompilationUnit -- No internal synchronization (Low-Medium)
**File:** `compilation_unit.h`
`CompilationUnit` holds `functions_`, `dict_`, `classDict_`, `classes_`, and `mangler_` with no internal locks. Methods like `register_function()`, `register_type()`, `find_function()`, `create_function()`, `_clear_python_cu()`, `unsafeRemoveMethod()`, and `drop_all_functions()` all mutate or read these containers without synchronization.

If a `CompilationUnit` is shared (via `shared_ptr`) and accessed concurrently, races are possible. In practice, compilation units are typically mutated during module construction/loading (single-threaded) and read during inference. The `Module` class does protect `register_buffer`/`register_parameter` with `register_mutex_`, but does not protect the underlying `CompilationUnit`.

**Severity: Low-Medium** -- Depends on usage patterns. The lack of synchronization is by design for performance, but concurrent `define()` + `find_function()` would race.

### 5. Module::register_mutex_ is per-Module but register_attribute() is unprotected (Low)
**File:** `module.h`, lines 126-148
`register_buffer()` and `register_parameter()` acquire `register_mutex_`, but `register_attribute()` (line 140) does not. If `register_attribute` is called concurrently with buffer/parameter registration on the same module, the `type()->addOrCheckAttribute()` and `_ivalue()->setAttr()` calls could race.

**Severity: Low** -- These registration methods are typically called during module setup, not at inference time.

**Recommendation:** Either protect `register_attribute()` with the same mutex, or document that all registration must happen single-threaded.

### 6. C10_DEFINE_bool (gflags) -- Safe for reads (Low)
**File:** `function_impl.cpp`, line 16-19
```cpp
C10_DEFINE_bool(
    torch_jit_do_not_store_optimized_graph,
    false, ...)
```
This global flag is read in `optimized_graph()` under `compile_mutex`. Writing to it concurrently with reads would be a race, but gflags values are typically set once at startup and then read. The read itself in `optimized_graph()` is protected by the lock anyway.

**Severity:** None in typical usage

### 7. No Python C API concerns
None of these files use Python C API directly. The `Method`, `Module`, and `Object` classes are C++ JIT infrastructure. Python bindings would be in separate files.

## Summary

The main concerns are the unprotected lazy-init patterns in `GraphFunction` (`ensure_defined()` and `getSchema()`). These are classic check-then-act races that become real data races under free-threading. The `CompilationUnit` and `Module` registration methods also lack complete synchronization, but are typically used single-threaded during setup.

The `compile_mutex` in `GraphFunction` already protects `optimized_graph()` and `get_executor()`, so extending its coverage to `ensure_defined()` and `getSchema()` would be the most natural fix.
