# Free-Threading Safety Audit: dynamo (part 1)

**Files audited:**
- `torch/csrc/dynamo/cache_entry.cpp`
- `torch/csrc/dynamo/cache_entry.h`
- `torch/csrc/dynamo/compiled_autograd.cpp`
- `torch/csrc/dynamo/compiled_autograd.h`
- `torch/csrc/dynamo/cpp_shim.cpp`
- `torch/csrc/dynamo/cpp_shim.h`
- `torch/csrc/dynamo/cpython_defs.h`
- `torch/csrc/dynamo/cpython_includes.h`
- `torch/csrc/dynamo/debug_macros.h`
- `torch/csrc/dynamo/eval_frame.h`
- `torch/csrc/dynamo/eval_frame_cpp.cpp`
- `torch/csrc/dynamo/eval_frame_cpp.h`
- `torch/csrc/dynamo/extra_state.cpp`
- `torch/csrc/dynamo/extra_state.h`
- `torch/csrc/dynamo/framelocals_mapping.cpp`

**Overall Risk: Medium-High**

---

## Issue 1 -- Global mutable `PyObject*`: `guard_error_hook`

- **Severity: Medium**
- **Location:** `eval_frame.c:21` (definition), `extra_state.cpp:173-176` (read), `eval_frame.c:674-678` (write via `set_guard_error_hook`)
- **Description:** `guard_error_hook` is a global `PyObject*` that can be set from Python via `set_guard_error_hook` and is read from `lookup()` in `extra_state.cpp` during guard evaluation. Under free-threading, one thread could be calling `set_guard_error_hook(None)` (clearing it) while another thread reads it in `lookup()`, causing a data race. The `Py_XSETREF` in `set_guard_error_hook` is not atomic with respect to the read in `extra_state.cpp`.
- **Suggested fix:** Use `_Py_atomic_store_ptr` / `_Py_atomic_load_ptr` for access, or protect with a lock. Since this is a rarely-changed hook, atomic load/store is sufficient.

## Issue 2 -- Global mutable `PyObject*`: `guard_complete_hook`

- **Severity: Medium**
- **Location:** `eval_frame.c:22` (definition), `eval_frame_cpp.cpp:564-571` (read), `eval_frame.c:682-696` (write via `set_guard_complete_hook`)
- **Description:** Same pattern as `guard_error_hook`. `guard_complete_hook` is read without synchronization in `dynamo__custom_eval_frame` and written from Python via `set_guard_complete_hook`. The write uses a non-atomic pointer swap pattern (`guard_complete_hook = Py_XNewRef(obj)`) while another thread may be simultaneously reading the old pointer value.
- **Suggested fix:** Use atomic pointer operations for reads and writes.

## Issue 3 -- Global mutable `bool`: `is_skip_guard_eval_unsafe`

- **Severity: Low**
- **Location:** `eval_frame.c:728` (definition), `eval_frame_cpp.cpp:542,583` (reads), `eval_frame.c:636-637` (write)
- **Description:** This bool is written from `set_skip_guard_eval_unsafe` and read from `dynamo__custom_eval_frame`. While bool reads/writes are likely atomic on most architectures, this is technically a data race per the C++ memory model. A torn read is unlikely but the lack of memory ordering guarantees could cause stale reads.
- **Suggested fix:** Use `std::atomic<bool>` or `_Py_atomic_store_int` / `_Py_atomic_load_int_relaxed`.

## Issue 4 -- Global mutable `PyObject*`: `bytecode_debugger_callback_obj`

- **Severity: Low**
- **Location:** `eval_frame_cpp.cpp:25` (definition), `eval_frame_cpp.cpp:58-72` (set/get), `eval_frame_cpp.cpp:464,472-473` (reads)
- **Description:** `bytecode_debugger_callback_obj` is a file-static raw `PyObject*` that is set via `set_bytecode_debugger_callback` and read via `get_bytecode_debugger_callback`, and also read directly in the eval frame path (line 464). These accesses are unsynchronized. Although this is a debugging feature unlikely to be toggled during hot paths, it is still a formal data race.
- **Suggested fix:** Use atomic pointer operations for reads and writes.

## Issue 5 -- Global mutable `std::unordered_set`: `breakpoint_code_objects`

- **Severity: Low**
- **Location:** `eval_frame_cpp.cpp:26` (definition), `eval_frame_cpp.cpp:75` (insert), `eval_frame_cpp.cpp:463` (read via `.count()`)
- **Description:** `breakpoint_code_objects` is a static `std::unordered_set<PyCodeObject*>` that can be inserted into from `register_breakpoint_code` while being read in the eval frame hot path via `.count()`. `std::unordered_set` is not thread-safe for concurrent read+write. A concurrent insert while another thread calls `.count()` can cause a crash or undefined behavior.
- **Suggested fix:** Protect with a mutex, or use an atomic flag to gate access, or use a thread-safe container. Since this is a debug-only feature, a simple mutex is fine.

## Issue 6 -- Global mutable `EvalFrameOverride`: `eval_frame_override`

- **Severity: Medium**
- **Location:** `eval_frame_cpp.cpp:342` (definition), `eval_frame_cpp.cpp:344-352` (get/set), `eval_frame_cpp.cpp:442` (read)
- **Description:** `eval_frame_override` is a global enum that is read in the eval frame hot path and written via `set_eval_frame_override`. This is a data race if one thread is inside `dynamo__custom_eval_frame` checking the override while another thread changes it. The enum is a small scalar type, but C++ memory model still requires synchronization.
- **Suggested fix:** Make this `std::atomic<EvalFrameOverride>`.

## Issue 7 -- Static lazy-init `PyObject*`: `random_module`

- **Severity: Medium**
- **Location:** `eval_frame_cpp.cpp:220-226`
- **Description:** Classic lazy-init pattern: `static PyObject* random_module = nullptr; if (random_module == nullptr) random_module = ...`. Two threads hitting this simultaneously will both see `nullptr`, both attempt the import, and one will leak the reference. The `py::module_::import` call itself is Python-safe but the store to the static is a data race.
- **Suggested fix:** Use `std::call_once`, or `_Py_atomic_compare_exchange_ptr`, or initialize eagerly during module init.

## Issue 8 -- Static lazy-init `std::optional<py::object>`: `convert_frame_get_fail_callback`

- **Severity: Medium**
- **Location:** `eval_frame_cpp.cpp:425-455`
- **Description:** `convert_frame_get_fail_callback` is a `static std::optional<py::object>` that is lazily initialized inside `dynamo__custom_eval_frame`. If two threads hit the `!convert_frame_get_fail_callback` branch simultaneously, both will attempt the import and assignment. `std::optional` assignment is not atomic, so this is a data race that can corrupt the optional's internal state. Additionally, the `atexit` registration would happen twice.
- **Suggested fix:** Use `std::call_once` or move initialization to module init.

## Issue 9 -- Global mutable `int32_t`: `c_recursion_limit`

- **Severity: Low**
- **Location:** `eval_frame_cpp.cpp:294` (definition), `eval_frame_cpp.cpp:296-305` (get/set), `eval_frame_cpp.cpp:313` (read inside `CRecursionLimitRAII`)
- **Description:** `c_recursion_limit` is a plain `int32_t` read in the eval frame path and written via `dynamo_set_c_recursion_limit`. This is a data race under free-threading.
- **Suggested fix:** Use `std::atomic<int32_t>`.

## Issue 10 -- Static mutable `bool`: `use_lru`

- **Severity: Low**
- **Location:** `extra_state.cpp:17` (definition), `extra_state.cpp:197,213` (reads), `extra_state.cpp:285-289` (write via `_set_lru_cache`)
- **Description:** The `use_lru` flag is an anonymous-namespace bool that controls cache insertion order. It can be written from Python via `_set_lru_cache` while being read during cache lookup/creation. Same class of bug as `is_skip_guard_eval_unsafe`.
- **Suggested fix:** Use `std::atomic<bool>`.

## Issue 11 -- Global `kActivePyCompilerInterface` (compiled_autograd)

- **Severity: Low-Medium**
- **Location:** `compiled_autograd.cpp:6-22`
- **Description:** `kActivePyCompilerInterface` is a `static std::unique_ptr<PyCompilerInterface>` that is set/cleared by `PyCompilerGuard` constructor/destructor. The guard asserts that the pointer is null before setting it, which means it is intended for single-threaded use. Under free-threading, if two threads try to use compiled autograd simultaneously, they could race on this global. The `unique_ptr` move and reset operations are not atomic. However, compiled autograd is typically invoked from the autograd engine which may have its own serialization, so practical risk depends on usage patterns.
- **Suggested fix:** Document the single-threaded requirement, or add a mutex if concurrent compiled autograd is desired.

## Issue 12 -- Global `Py_ssize_t extra_index`

- **Severity: Low**
- **Location:** `extra_state.cpp:20` (definition), `extra_state.h:35` (extern), `eval_frame.c:758` (initialization)
- **Description:** `extra_index` is set once during module init (`torch_c_dynamo_eval_frame_init`) and then read during all subsequent operations. The init-once-read-many pattern is safe if module init happens before any concurrent use, which is guaranteed by Python's import machinery. However, there is no memory fence between init and first use from other threads. On x86 this is benign; on ARM it could theoretically be stale.
- **Suggested fix:** Low priority. Could use `std::atomic` with relaxed store at init and acquire load at use for correctness on all architectures, but practically safe.

## Issue 13 -- `_PyInterpreterState_SetEvalFrameFunc` in `debug_macros.h`

- **Severity: Low**
- **Location:** `debug_macros.h:58-65`
- **Description:** The `_debug_set_eval_frame` inline function sets the interpreter-wide eval frame function. Under free-threading, changing the interpreter-global eval frame callback from one thread affects all threads. This is inherently racy -- another thread could be in the middle of checking/using the eval frame function while it changes. However, this function appears to only be used in the `INSPECT` debug macro, so practical risk is minimal.
- **Suggested fix:** Acceptable for debug-only code. Document that `INSPECT` is not thread-safe.

## Notes

- **`eval_frame_callback_key` (TSS):** The eval frame callback is stored per-thread via `Py_tss_*` (Thread Specific Storage), which is inherently thread-safe. This is a good pattern.
- **`ExtraState` on code objects:** Extra state is attached to code objects via `_PyCode_SetExtra`/`_PyCode_GetExtra`. Under free-threading, code objects can be shared across threads, so the extra state could be accessed concurrently. CPython 3.13t does provide some internal locking for code extra data, but the `ExtraState` itself (cache_entry_list, frame_state, strategy) has no internal synchronization. If two threads execute the same code object simultaneously and both trigger compilation, they could race on `init_and_set_extra_state` (double init) or concurrent cache list modification. This is a systemic concern that likely needs a per-ExtraState lock or critical section.
- **`compiled_autograd.h`:** The header is primarily template/struct definitions with no global mutable state. The `AutogradCompilerCall`, `CompiledNodeArgs`, `SwapSavedVariables`, etc. are all per-invocation stack-local objects, which are thread-safe.
- **`cpp_shim.cpp`:** Pure RAII wrappers around `RecordFunction`, no global state. Thread-safe.
- **`cpython_defs.h`, `cpython_includes.h`:** Macro definitions and type declarations only. No mutable state.
- **`framelocals_mapping.cpp`/`.h`:** `FrameLocalsMapping` is constructed per-frame and used locally. No global state. Thread-safe.
