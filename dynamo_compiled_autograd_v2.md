# Free-Threading Audit: dynamo compiled autograd (v2)

## Files Audited

- `torch/csrc/dynamo/python_compiled_autograd.cpp`
- `torch/csrc/dynamo/python_compiled_autograd.h`
- `torch/csrc/dynamo/compiled_autograd.cpp`
- `torch/csrc/dynamo/compiled_autograd.h`

## Architecture Summary

Compiled autograd replaces the standard autograd engine by converting the
backward autograd graph into an FX graph that can be `torch.compile`d. The key
pieces of mutable shared state are:

1. **`the_autograd_compiler`** -- file-static `PyObject*` (anonymous namespace).
   Set by `set_autograd_compiler()`, read by `_compiled_autograd_impl()` on
   cache miss to instantiate a Python compiler. Owns a strong reference
   (`Py_INCREF` on set, prior value returned to Python for decref).

2. **`CacheNode` tree** -- rooted at `CacheNode::root()` (function-local
   static). A shadow graph that caches compiled FX graphs. Each internal node
   has `next` (`unordered_map<CacheKey, unique_ptr<CacheNode>>`). Leaf nodes
   store `runtime_wrapper` and `compiled_fn` (`THPObjectPtr`s) plus
   `expected_sizes`.

3. **`active_rstate`** -- file-static `RuntimeState*`. Set by
   `RuntimeStateGuard` in `compiled_autograd()`, read by the
   `call_cpp_tensor_pre_hooks` Python C method during compiled graph execution.

4. **`python_verbose_logger`** -- file-static `PyObject*` (anonymous
   namespace). A **borrowed** reference (no `Py_INCREF`). Set by
   `set_verbose_logger()`, read by `VerboseLogger::maybe_create()`.

5. **`default_dyn_type_int`** -- file-static `int` (anonymous namespace). Set
   by `set_autograd_compiler()`, read by `get_default_dyn_type()`.

6. **`kActivePyCompilerInterface`** -- file-static
   `unique_ptr<PyCompilerInterface>` in `compiled_autograd.cpp`. Set/cleared by
   `PyCompilerGuard` during cache-miss compilation. Read by
   `getPyCompilerInterface()` from various autograd node implementations in
   libtorch_cpu.

**Concurrency model.** The `compiled_autograd()` entry point acquires a
file-static `std::mutex` (via `try_lock` -- fails immediately if contended)
then acquires the GIL. All cache traversal, compilation, and compiled graph
execution happen under both the mutex and the GIL. This serializes backward
passes effectively.

However, the five Python C methods exported by this module --
`set_autograd_compiler`, `clear_cache`, `is_cache_empty`,
`set_verbose_logger`, `call_cpp_tensor_pre_hooks` -- are **not** protected by
the mutex. On free-threaded Python 3.14t, the module declares
`Py_MOD_GIL_NOT_USED`, meaning CPython will not automatically acquire the GIL
before calling any of these methods. They can therefore race with the
mutex-protected backward path and with each other.

## SEVERE Issues

### S1. `CacheNode` tree: `clear_cache` races with backward traversal

- **Tier:** Tier 1 (can be triggered from any thread, e.g. a data loader
  thread calling into Python code that clears the cache)
- **Shared state:** The entire `CacheNode` tree rooted at
  `CacheNode::root()`.
- **Writer(s):** `clear_cache()` (line 637) calls
  `CacheNode::root()->clear()`, which destroys all child nodes by clearing
  `next` (an `unordered_map` of `unique_ptr<CacheNode>`), `key_storage`,
  `expected_sizes`, and releases `runtime_wrapper`/`compiled_fn`.
- **Reader(s):** `_compiled_autograd_impl()` (line 949) traverses the tree via
  `cache = cache->lookup(key)`, obtaining raw pointers to child `CacheNode`s.
  Later (line 976) it calls `cache->check_dynamic_sizes()` and (lines
  1150-1157) reads `cache->runtime_wrapper` and `cache->compiled_fn`. Also,
  `compiled_autograd()` (lines 1238-1246) uses `cache->runtime_wrapper` and
  `cache->compiled_fn` after `_compiled_autograd_impl` returns.
- **Race scenario:** Thread A is inside `compiled_autograd()` holding the
  mutex and GIL, has traversed several cache nodes, and holds a raw pointer
  `cache` to a non-root `CacheNode`. Thread B calls `clear_cache()` from
  Python. Because `clear_cache` is `METH_NOARGS` on a `Py_MOD_GIL_NOT_USED`
  module, Thread B enters without acquiring the GIL or the compiled autograd
  mutex. `CacheNode::root()->clear()` destroys all children via
  `next.clear()`, which invokes `unique_ptr<CacheNode>` destructors. Thread
  A's `cache` pointer is now dangling. Thread A proceeds to dereference it --
  use-after-free, likely a crash.
- **Suggested fix:** `clear_cache` must acquire the compiled autograd mutex
  before clearing. Alternatively, remove `Py_MOD_GIL_NOT_USED` so the GIL
  serializes all Python C method calls (the GIL alone is sufficient if
  `compiled_autograd()` also holds the GIL for its entire duration, which it
  does).

### S2. `the_autograd_compiler` pointer: unsynchronized read vs. write

- **Tier:** Tier 1 (any thread can call `set_autograd_compiler`)
- **Shared state:** `the_autograd_compiler` (file-static `PyObject*`, line 54)
- **Writer(s):** `set_autograd_compiler()` (line 1252) writes the pointer:
  either sets it to `nullptr` (disable) or to a new `PyObject*` with
  `Py_INCREF` (enable). No lock is held.
- **Reader(s):** `_compiled_autograd_impl()` (line 980) reads the pointer to
  call `PyObject_CallNoArgs(the_autograd_compiler)`. This runs under the
  compiled autograd mutex and GIL, but the writer does not hold either.
- **Race scenario:** Thread A is in `_compiled_autograd_impl` about to call
  `PyObject_CallNoArgs(the_autograd_compiler)`. Thread B calls
  `set_autograd_compiler(None, 0)` from Python. On 3.14t, Thread B does not
  need the GIL to enter the function. Thread B sets
  `the_autograd_compiler = nullptr` and returns the prior `PyObject*` to the
  caller, which decrefs it. If Thread A has already loaded the pointer value
  but not yet completed the call, it calls into a freed object. Even on
  architectures where pointer writes are atomic, the refcount management is
  unsynchronized: the object can be freed before Thread A's call completes.
- **Suggested fix:** Have `set_autograd_compiler` acquire the compiled
  autograd mutex before modifying the pointer. Or remove `Py_MOD_GIL_NOT_USED`
  so the GIL serializes access.

### S3. `active_rstate` global pointer: unsynchronized read from `call_cpp_tensor_pre_hooks`

- **Tier:** Tier 2 (only relevant if `call_cpp_tensor_pre_hooks` is called
  from a thread other than the one running the compiled backward)
- **Shared state:** `active_rstate` (file-static `RuntimeState*`, line 96)
- **Writer(s):** `RuntimeStateGuard` constructor (line 99) sets
  `active_rstate = _state.get()`; destructor (line 107) sets
  `active_rstate = nullptr`. These run inside `compiled_autograd()` under the
  mutex.
- **Reader(s):** `call_cpp_tensor_pre_hooks()` (line 113) reads
  `active_rstate` and dereferences it. This is a `METH_VARARGS` method on a
  `Py_MOD_GIL_NOT_USED` module, callable without the GIL from any thread.
- **Race scenario:** Thread A runs the compiled backward, which calls the
  compiled FX graph. The FX graph calls into Python, which eventually calls
  `call_cpp_tensor_pre_hooks`. Meanwhile Thread B triggers a different code
  path that also calls `call_cpp_tensor_pre_hooks` (unlikely but possible
  since it is a publicly exported method). Thread B reads `active_rstate`
  which is either `nullptr` (assert fires) or points to Thread A's
  `RuntimeState` (data race on the `RuntimeState` internals). More
  realistically, if `compiled_autograd` releases the GIL during FX graph
  execution (it does not currently, but `pybind11::gil_scoped_acquire` is at
  function scope), another thread could call `call_cpp_tensor_pre_hooks`.
  The write (pointer store) and read are both on a plain (non-atomic) pointer
  -- this is a C++ data race (undefined behavior).
- **Suggested fix:** Make `active_rstate` thread-local, or pass the
  `RuntimeState` as a Python capsule argument rather than using a global.

### S4. `python_verbose_logger` borrowed reference: use-after-free

- **Tier:** Tier 1 (any thread can call `set_verbose_logger`)
- **Shared state:** `python_verbose_logger` (file-static `PyObject*`, line 56)
  -- stored as a **borrowed reference** (no `Py_INCREF`).
- **Writer(s):** `set_verbose_logger()` (line 652) stores the raw `PyObject*`
  directly, or sets it to `nullptr`. No reference count increment is performed.
- **Reader(s):** `VerboseLogger::maybe_create()` (line 398) reads the pointer.
  `PythonLogger::log()` (line 370) dereferences it. Both run inside the
  compiled backward path under the mutex and GIL.
- **Race scenario 1 (free-threading):** Thread A is inside a compiled backward,
  has captured `python_verbose_logger` into a `VerboseLogger` instance (line
  402). Thread B calls `set_verbose_logger(None)` which sets
  `python_verbose_logger = nullptr`. The Python logger object may be
  garbage-collected if the caller held the only reference (since no `Py_INCREF`
  was done). Thread A later calls `vlogger->log()` which dereferences the
  freed object.
- **Race scenario 2 (single-threaded):** Even without concurrency, the
  borrowed reference is fragile: if the Python caller of `set_verbose_logger`
  drops its reference, the stored pointer dangles. This is not a threading
  issue per se, but free-threading makes it much more likely to manifest.
- **Suggested fix:** `Py_XINCREF` the logger in `set_verbose_logger` and
  `Py_XDECREF` the old one. Use `THPObjectPtr` for the global. Protect reads
  with the mutex or GIL.

## Significant Issues

### G1. `kActivePyCompilerInterface` unprotected global `unique_ptr`

- **Tier:** Tier 2 (requires concurrent compiled autograd use)
- **Shared state:** `kActivePyCompilerInterface` (file-static
  `unique_ptr<PyCompilerInterface>` in `compiled_autograd.cpp`, line 6).
- **Writer(s):** `PyCompilerGuard` constructor (line 16) moves a value in;
  destructor (line 21) resets to `nullptr`. Called from
  `_compiled_autograd_impl` during cache-miss compilation.
- **Reader(s):** `getPyCompilerInterface()` (line 8) returns a const reference
  to the `unique_ptr`. Called from `SwapSavedVariables::before(SavedVariable&)`
  (compiled_autograd.h line 819) during tracing, and from various autograd node
  implementations in libtorch_cpu (`custom_function.h:348`,
  `accumulate_grad.cpp:165,174`, `tensor.cpp:76,94,264`).
- **Race scenario:** Currently, all callers of `getPyCompilerInterface` are
  reached only during the cache-miss tracing phase, which runs under the
  compiled autograd mutex. The protection is therefore implicit. However,
  `getPyCompilerInterface()` is `TORCH_API` (public) and performs no
  synchronization. If a future code change calls it from outside the
  mutex-protected path, or if the mutex is relaxed for concurrent backwards,
  two threads could race on the `unique_ptr` (one writing via
  `PyCompilerGuard`, another reading via `getPyCompilerInterface`). This would
  corrupt the `unique_ptr` internals.
- **Suggested fix:** Make `kActivePyCompilerInterface` thread-local (it is
  semantically per-compilation-session), or protect it with a dedicated mutex
  or the compiled autograd mutex.

### G2. `set_autograd_compiler` non-atomic read-modify-write sequence

- **Tier:** Tier 2 (requires concurrent `set_autograd_compiler` calls)
- **Shared state:** `the_autograd_compiler`, `default_dyn_type_int`,
  `Engine::the_compiled_autograd` (atomic in engine.cpp).
- **Writer(s):** `set_autograd_compiler()` performs a multi-step sequence:
  reads `the_autograd_compiler` (line 1261), reads `default_dyn_type_int`
  (line 1262), writes `default_dyn_type_int` (line 1263), conditionally writes
  `the_autograd_compiler` (lines 1265/1269), and calls
  `Engine::set_compiled_autograd` (lines 1266/1270). No lock is held.
- **Reader(s):** Another concurrent `set_autograd_compiler` call, or the
  compiled backward path.
- **Race scenario:** Two threads call `set_autograd_compiler` concurrently.
  Thread A reads `prior_compiler = the_autograd_compiler`, capturing a pointer.
  Thread B overwrites `the_autograd_compiler` and `Py_INCREF`s its new value.
  Thread A then overwrites with its own value. The prior captured by Thread A
  is stale -- it may point to an object that Thread B has already returned for
  decref. The refcount bookkeeping becomes corrupted, leading to either a leak
  or a double-free.
- **Suggested fix:** Protect the entire `set_autograd_compiler` function body
  with the compiled autograd mutex.

### G3. `default_dyn_type_int` non-atomic int

- **Tier:** Tier 1 (written by any thread calling `set_autograd_compiler`,
  read during backward)
- **Shared state:** `default_dyn_type_int` (file-static `int`, line 55)
- **Writer(s):** `set_autograd_compiler()` (line 1263)
- **Reader(s):** `get_default_dyn_type()` (line 883) during
  `_compiled_autograd_impl`.
- **Race scenario:** Thread A is in `_compiled_autograd_impl` calling
  `get_default_dyn_type()` while Thread B calls `set_autograd_compiler`. The
  read/write is on a plain `int` with no synchronization -- a C++ data race
  (undefined behavior). In practice the consequence is benign (wrong dynamic
  type for one compilation), but it is technically UB.
- **Suggested fix:** Use `std::atomic<int>`.

### G4. `CacheNode::check_dynamic_sizes` mutates shared state without external protection

- **Tier:** Tier 2 (requires concurrent backwards, currently blocked by mutex)
- **Shared state:** `CacheNode::expected_sizes`, `runtime_wrapper`,
  `compiled_fn`.
- **Writer(s):** `check_dynamic_sizes()` (line 509) modifies `expected_sizes`
  and may null out `runtime_wrapper`/`compiled_fn` on a shape-change cache
  miss.
- **Reader(s):** `compiled_autograd()` reads `cache->runtime_wrapper` and
  `cache->compiled_fn` after the check. `is_cache_empty()` reads
  `compiled_fn` and `next.empty()`.
- **Race scenario:** Currently the compiled autograd mutex serializes backward
  calls, so two threads cannot be in `check_dynamic_sizes` on the same
  `CacheNode` simultaneously. However, `is_cache_empty()` (line 643) has no
  synchronization and reads `compiled_fn` and `next` without any lock. If
  Thread A is inside `check_dynamic_sizes` nulling out `compiled_fn` (line
  567-568) while Thread B calls `is_cache_empty`, the read of `compiled_fn`
  races with the write. This is a data race on the `THPObjectPtr` internals.
- **Suggested fix:** `is_cache_empty` should acquire the compiled autograd
  mutex (or at minimum the GIL, since `THPObjectPtr` is ultimately a
  `PyObject*`).

## Minor Issues

### M1. `Py_MOD_GIL_NOT_USED` is the root cause of S1-S4

- **Tier:** Tier 1
- **Shared state:** All module-level functions.
- **Race scenario:** By declaring `Py_MOD_GIL_NOT_USED`, all five exported C
  methods can be called without the GIL on 3.14t. None of them have adequate
  internal synchronization for concurrent use. Removing this declaration would
  cause CPython to acquire the GIL before entering any module method, which
  combined with the `pybind11::gil_scoped_acquire` in `compiled_autograd()`
  would serialize all relevant accesses and eliminate S1-S4 without any other
  code changes.
- **Suggested fix:** Remove `Py_MOD_GIL_NOT_USED` until the module methods
  have proper internal synchronization. This is the highest-leverage single
  fix.

### M2. Static local `PyObject*` from `PyUnicode_InternFromString` in hot path

- **Tier:** Tier 2
- **Shared state:** `static PyObject* method_name` locals in
  `call_begin_capture` (line 803), `call_end_capture` (line 857),
  `ClosingTHPObjectPtr::~ClosingTHPObjectPtr` (line 874), and the `static bool
  flag` lambda (line 1077).
- **Race scenario:** C++11 guarantees thread-safe initialization of
  function-local statics. However, the `static bool flag` lambda at line 1077
  calls `bind_function` which calls into Python. If another thread is blocked
  on the same static initialization guard while Python code tries to re-enter
  this initialization path, a deadlock could occur. The
  `PyUnicode_InternFromString` calls are safe as they are idempotent, but
  they assume the GIL is held (which it is under current usage since
  `compiled_autograd()` acquires it).
- **Suggested fix:** Move one-time initialization to module init, or to a
  `std::call_once` with the compiled autograd mutex held.

### M3. `CacheNode` destructor leak on interpreter shutdown

- **Tier:** N/A (shutdown-only)
- **Shared state:** `CacheNode::runtime_wrapper`, `CacheNode::compiled_fn`.
- **Race scenario:** `~CacheNode()` (lines 497-503) checks
  `Py_IsInitialized()` and leaks (`release()`) if the interpreter is
  finalizing. Under free-threading, a thread could still be executing compiled
  autograd when the interpreter starts finalizing. `Py_IsInitialized` could
  return true on entry to the destructor but become false by the time
  `Py_DECREF` runs, causing a crash.
- **Suggested fix:** Register an atexit handler to clear the cache before
  interpreter finalization.

### M4. `CompiledAutogradThreadingDebugCheck` TOCTOU in engine.cpp

- **Tier:** Tier 2
- **Shared state:** `num_threads_in_compiled_autograd` (atomic int32 in
  engine.cpp).
- **Race scenario:** The check
  `num_threads_in_compiled_autograd.load() == 0` is a TOCTOU race: two
  threads could both see 0, both proceed past the check, and both try to
  acquire the compiled autograd mutex. The mutex (`try_lock`) catches the
  actual conflict, so this is merely a misleading error message, not a
  correctness issue. Additionally, `_thread_check.release()` is called before
  entering `compiled_autograd`, so the counter is decremented before the
  mutex is acquired -- a third thread could slip past the check.
- **Suggested fix:** This is informational only; the mutex provides the real
  protection. Consider removing the debug check or making it accurate.
