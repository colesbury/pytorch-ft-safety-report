# Free-Threading Safety Audit: jit/frontend (Part 1)

## Overall Risk: LOW

Most files in this batch operate on JIT IR graphs via pure transformations with no global/static mutable state and no Python C API calls. The one notable concern is the lazy-initialization pattern in `builtin_functions.cpp`, which is already protected by a recursive mutex and is safe. The `error_report.cpp` thread-local shared_ptr pattern is also already designed for cross-thread safety. The `concrete_module_type.cpp` file calls into Python but is only used during compilation, which holds the GIL today and is expected to continue doing so.

---

## Issues

### Issue 1 — Lazy init of static BuiltinFunctionRegistry returns reference to interior data

- **Category:** Lazy init / static mutable C++ state
- **Severity:** Low
- **Confidence:** High
- **File:** `torch/csrc/jit/frontend/builtin_functions.cpp`
- **Lines:** 94-188
- **Description:**
  `getAllBuiltinFunctionsFor` uses a function-local `static BuiltinFunctionRegistry registry` with a `std::recursive_mutex` to guard lazy initialization and lookups. The C++ runtime guarantees the static is constructed exactly once. The recursive mutex correctly prevents deadlock during re-entrant initialization. However, the function returns a `const std::vector<Function*>&` — a reference into the registry's internal `builtins_by_name_` map. If a caller holds this reference across threads while another thread is still in the `INITIALIZING` phase (which can mutate `builtins_by_name_` via `loadSource`), the reference could observe a partially-constructed map. In practice this is safe because: (a) the mutex is held during init, so concurrent callers block until `INITIALIZED`, and (b) the `INITIALIZING` early-return path returns the static `empty` vector, not the map. After initialization the map is immutable, so concurrent reads of the returned reference are safe.
- **Fix:** No fix required. The existing locking is correct. The pattern is safe because the map is effectively immutable after initialization, and all callers either block until init completes or receive the empty sentinel. Consider adding a comment noting this invariant for future maintainers.

### Issue 2 — Static thread_local shared_ptr<Calls> in error_report.cpp

- **Category:** Static mutable C++ state / thread_local
- **Severity:** Low
- **Confidence:** High
- **File:** `torch/csrc/jit/frontend/error_report.cpp`
- **Lines:** 33-34
- **Description:**
  A `thread_local std::shared_ptr<ErrorReport::Calls>` is used to maintain a per-thread call stack. The code already accounts for the cross-thread reference scenario (documented in the comment at lines 7-31): `CallStack` stores a copy of the `shared_ptr` so it pops from the correct thread's stack even if the destructor runs on a different thread. The `Calls` class itself uses a `std::mutex` for all operations. This is already designed for cross-thread safety. Under free-threading, `thread_local` remains per-thread, so the `shared_ptr` itself is not shared. The `Calls` object may be shared across threads (as documented), but its mutex protects all accesses.
- **Fix:** No fix required. The existing design with mutex-protected `Calls` and per-thread `shared_ptr` is correct for free-threading.

### Issue 3 — Python C API calls in ConcreteModuleTypeBuilder::createTypeFromThis without critical section

- **Category:** Python C API on shared objects without Py_BEGIN_CRITICAL_SECTION
- **Severity:** Low
- **Confidence:** Medium
- **File:** `torch/csrc/jit/frontend/concrete_module_type.cpp`
- **Lines:** 10-16
- **Description:**
  `createTypeFromThis()` calls `py::module::import("torch._jit_internal").attr("_qualified_name")(pyClass_)` and `py::cast<std::string>(pyQualName)`. These Python C API calls operate on Python objects (`pyClass_`, the import result, the qualified name). Under free-threading, if two threads concurrently call this on the same `ConcreteModuleTypeBuilder` instance sharing `pyClass_`, there could be races on the Python object's reference count or internal state. However, this code path is only invoked during TorchScript compilation (`ConcreteModuleType::build()`), which is called from Python-level `torch.jit.script` / `torch.jit.trace`. These entry points would typically hold the GIL or be serialized. The `ConcreteModuleTypeBuilder` itself is a short-lived builder object, not shared across threads.
- **Fix:** No immediate fix required. If TorchScript compilation is ever parallelized without GIL protection, these Python calls would need `Py_BEGIN_CRITICAL_SECTION` or equivalent synchronization. For now, the usage pattern is safe.

### Issue 4 — ConcreteModuleTypeBuilder::addConstant calls tryToInferType/toIValue on Python objects

- **Category:** Python C API on shared objects
- **Severity:** Low
- **Confidence:** Medium
- **File:** `torch/csrc/jit/frontend/concrete_module_type.cpp`
- **Lines:** 224-238
- **Description:**
  `addConstant(std::string name, py::object value)` calls `tryToInferType(value)` and `toIValue(std::move(value), match.type())`, which inspect Python objects. Same situation as Issue 3 — these are called during the builder phase of TorchScript compilation, which is serialized by the GIL in current usage.
- **Fix:** No immediate fix required. Same caveat as Issue 3.

---

## Files With No Issues

The following files contain no free-threading safety concerns:

- **`jit/frontend/builtin_functions.h`** — Declaration only, no mutable state.
- **`jit/frontend/canonicalize_modified_loop.cpp`** — Pure JIT IR graph transformation, no global/static state, no Python C API.
- **`jit/frontend/canonicalize_modified_loop.h`** — Declaration only.
- **`jit/frontend/concrete_module_type.h`** — Class declarations with no static state. All mutable state is per-instance.
- **`jit/frontend/convert_to_ssa.cpp`** — Pure JIT IR graph transformation using local structs (`ControlFlowLoadStores`, `EraseLoadStores`, `LoopContinuations`). All state is stack-local or per-instance. No global/static mutable state, no Python C API.
- **`jit/frontend/convert_to_ssa.h`** — Declaration only.
- **`jit/frontend/edit_distance.cpp`** — Pure algorithmic function (`ComputeEditDistance`) with only stack-local state. No global/static mutable state.
- **`jit/frontend/edit_distance.h`** — Declaration only.
- **`jit/frontend/error_report.h`** — Class declarations. The `Calls` inner class is mutex-protected (see Issue 2).
- **`jit/frontend/exit_transforms.cpp`** — Pure JIT IR graph transformation using local structs (`ExitTransformer`). All state is per-instance. Static helper functions operate only on passed arguments. No global/static mutable state, no Python C API.
- **`jit/frontend/exit_transforms.h`** — Declaration only.
- **`jit/frontend/function_schema_parser.cpp`** — Parsing functions (`parseSchema`, `parseName`, `parseSchemaOrName`) that create stack-local `SchemaParser` instances. No global/static mutable state. `c10::getStringToDtypeMap()` at line 227 returns a const reference to a static map that is populated at static init time and is immutable thereafter.
