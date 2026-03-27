# Deep Free-Threading Audit: dynamo compiled autograd

## Architecture Summary

Compiled autograd replaces the standard autograd engine by converting the autograd
graph into an FX graph that can be `torch.compiled`. The key components:

**The CacheNode tree** (`CacheNode::root()` is a static singleton): A shadow graph
that caches compiled FX graphs. Each autograd node produces a `CacheKey` from its
constant properties; the key is used to traverse `CacheNode::next` (an
`unordered_map<CacheKey, unique_ptr<CacheNode>>`). A leaf `CacheNode` stores
`runtime_wrapper` and `compiled_fn` (`THPObjectPtr`s).

**The compiler interface**: `the_autograd_compiler` (file-static `PyObject*`) is
the Python callable that creates an `AutogradCompilerInstance`. It is set/cleared
by `set_autograd_compiler()`. When a cache miss occurs, it is called to produce a
`py_compiler` instance that traces the backward graph.

**The mutex**: `compiled_autograd()` holds a file-static `std::mutex` for its
entire duration (cache lookup + possible compilation + execution). It uses
`try_lock` and fails immediately if the lock is contended, meaning only one
compiled backward can run at a time. The GIL is acquired after the mutex.

**`active_rstate`**: A file-static raw pointer to a `RuntimeState` that holds
`cpp_tensor_pre_hooks`. It is set by `RuntimeStateGuard` in `compiled_autograd()`
and read by the `call_cpp_tensor_pre_hooks` Python C method.

**`kActivePyCompilerInterface`**: A static `unique_ptr<PyCompilerInterface>` in
`compiled_autograd.cpp`, set/cleared by `PyCompilerGuard` during cache-miss
compilation. Used by `SwapSavedVariables` to call back into Python.

**Module GIL declaration**: The module init sets `Py_MOD_GIL_NOT_USED`, declaring
to the free-threaded runtime that this extension does not require the GIL. This
means all `METH_VARARGS`/`METH_NOARGS` functions exported by this module
(`set_autograd_compiler`, `clear_cache`, `is_cache_empty`, `set_verbose_logger`,
`call_cpp_tensor_pre_hooks`) can be called without the GIL held by the caller on
Python 3.14t.

## SEVERE Issues

### S1. `the_autograd_compiler` borrowed reference: use-after-free on concurrent `set_autograd_compiler`

- **Shared state:** `the_autograd_compiler` (file-static `PyObject*`)
- **Writer(s):** `set_autograd_compiler()` -- sets the pointer and manages refcounts. Called from Python (no GIL guaranteed on 3.14t since the module declares `Py_MOD_GIL_NOT_USED`).
- **Reader(s):** `_compiled_autograd_impl()` -- reads the pointer at line 980: `PyObject_CallNoArgs(the_autograd_compiler)`. Also `Engine::execute` reads the compiled_autograd function pointer, which is only valid while `the_autograd_compiler` is valid.
- **Race scenario:** Thread A is inside `_compiled_autograd_impl` (holding the compiled autograd mutex and GIL) and calls `PyObject_CallNoArgs(the_autograd_compiler)`. Thread B calls `set_autograd_compiler(None, 0)` from Python. On 3.14t, Thread B does NOT need the GIL to enter `set_autograd_compiler` (it's a `METH_VARARGS` on a `Py_MOD_GIL_NOT_USED` module). Thread B sets `the_autograd_compiler = nullptr` and the prior `PyObject*` is returned to the caller who decrefs it. If Thread A's `PyObject_CallNoArgs` is in flight or about to execute, it operates on a dangling pointer. Even with the GIL, the write to the pointer itself is a plain store (not atomic) so Thread B's store could tear on architectures with relaxed memory ordering.
- **Suggested fix:** Protect `the_autograd_compiler` with the compiled autograd mutex, OR use `std::atomic<PyObject*>` with proper `Py_INCREF`/`Py_DECREF` bracketing, OR (simplest) remove `Py_MOD_GIL_NOT_USED` and require GIL for all module methods. If keeping nogil, `set_autograd_compiler` must acquire the compiled autograd mutex so it cannot race with `compiled_autograd()`.

### S2. `active_rstate` global pointer: data race between writer and `call_cpp_tensor_pre_hooks`

- **Shared state:** `active_rstate` (file-static `RuntimeState*`)
- **Writer(s):** `RuntimeStateGuard` constructor/destructor in `compiled_autograd()`, which runs under the compiled autograd mutex + GIL.
- **Reader(s):** `call_cpp_tensor_pre_hooks()` -- a `METH_VARARGS` Python C method, callable from any thread. On 3.14t with `Py_MOD_GIL_NOT_USED`, no GIL is held on entry. It reads `active_rstate` and dereferences it.
- **Race scenario:** Thread A runs the compiled backward, which calls the compiled FX graph. Inside that graph, `call_cpp_tensor_pre_hooks` is called, reading `active_rstate`. Meanwhile, Thread B starts or finishes a different compiled backward (or the same thread's backward finishes from an async callback), and `RuntimeStateGuard` destructor sets `active_rstate = nullptr`. Thread A dereferences `nullptr` or a pointer to a destroyed `RuntimeState`. Even if Thread B cannot enter `compiled_autograd` concurrently (due to the mutex), the compiled FX graph execution happens *after* `_compiled_autograd_impl` returns but *while* the mutex is still held -- however `call_cpp_tensor_pre_hooks` could also be called from user code outside the compiled backward entirely.
- **Suggested fix:** Either (a) make `call_cpp_tensor_pre_hooks` require the GIL (remove `Py_MOD_GIL_NOT_USED`), or (b) pass the `RuntimeState` as a Python object argument to the hooks rather than using a global, or (c) use thread-local storage for `active_rstate`.

### S3. `CacheNode` tree: concurrent `clear_cache` during backward traversal

- **Shared state:** The `CacheNode` tree rooted at `CacheNode::root()` (static singleton).
- **Writer(s):** `clear_cache()` -- a `METH_NOARGS` Python C method callable without GIL on 3.14t. Calls `CacheNode::root()->clear()` which destroys the entire tree (`next.clear()`, `key_storage.clear()`, releases `runtime_wrapper` and `compiled_fn`).
- **Reader(s):** `_compiled_autograd_impl()` traverses the tree via `cache->lookup(key)` and later reads `cache->runtime_wrapper` and `cache->compiled_fn`.
- **Race scenario:** Thread A is inside `_compiled_autograd_impl`, has traversed several `CacheNode`s, and holds a raw pointer `cache` to a non-root node. Thread B calls `clear_cache()`. Since `clear_cache` is `METH_NOARGS` and the module is `Py_MOD_GIL_NOT_USED`, Thread B enters without any lock. `CacheNode::root()->clear()` destroys all child nodes. Thread A's `cache` pointer is now dangling. Thread A proceeds to call `cache->check_dynamic_sizes()` or read `cache->compiled_fn` -- use-after-free.
- **Suggested fix:** `clear_cache` must acquire the compiled autograd mutex. Alternatively, require the GIL for this module.

### S4. `python_verbose_logger` borrowed reference: use-after-free

- **Shared state:** `python_verbose_logger` (file-static `PyObject*`, a borrowed reference -- no incref is done).
- **Writer(s):** `set_verbose_logger()` -- stores the raw pointer without incrementing the reference count. Called from Python.
- **Reader(s):** `VerboseLogger::maybe_create()` reads the pointer. `PythonLogger::log()` dereferences it to call methods on it.
- **Race scenario:** Thread A is inside a compiled backward (holding mutex + GIL), calls `VerboseLogger::maybe_create()` which captures `python_verbose_logger` into a `VerboseLogger`. Thread B calls `set_verbose_logger(None)` which sets `python_verbose_logger = nullptr`. The Python logger object's last reference may be dropped (e.g., the Python caller had the only reference). The object is GC'd. Thread A later calls `vlogger->log()` which dereferences `logger_` -- use-after-free. This can also happen within a single thread if the caller of `set_verbose_logger` drops its reference to the Python logger object after calling the function; the stored borrowed reference would dangle. On 3.14t without GIL, this is even worse because `set_verbose_logger` can race without synchronization.
- **Suggested fix:** `Py_INCREF` the logger in `set_verbose_logger`, `Py_XDECREF` the old one, and `Py_XDECREF` in a cleanup path (or use `THPObjectPtr`). Also protect the read with the mutex or GIL.

### S5. `kActivePyCompilerInterface`: concurrent access from multiple threads

- **Shared state:** `kActivePyCompilerInterface` (file-static `unique_ptr<PyCompilerInterface>` in `compiled_autograd.cpp`).
- **Writer(s):** `PyCompilerGuard` constructor/destructor, called from `_compiled_autograd_impl` during cache-miss compilation.
- **Reader(s):** `getPyCompilerInterface()` is called from `SwapSavedVariables::before(SavedVariable&)` which runs during the cache-miss tracing phase. Also accessible from any code in `libtorch_cpu` that includes the header.
- **Race scenario:** This global is not protected by any mutex. While one backward is compiling (cache miss), if any other code in libtorch_cpu calls `getPyCompilerInterface()` from another thread, it reads a non-atomic `unique_ptr` concurrently with potential writes. The compiled autograd mutex in `python_compiled_autograd.cpp` does not protect this global in `compiled_autograd.cpp`. Although the current call pattern may make this unlikely (only called during compilation), the interface is public (`TORCH_API`) and the protection is implicit/accidental.
- **Suggested fix:** Either make `kActivePyCompilerInterface` thread-local, or protect it with a mutex, or document that it must only be accessed while the compiled autograd mutex is held.

## Significant Issues

### G1. `set_autograd_compiler` non-atomic read-modify-write of `the_autograd_compiler` reference count

- **Shared state:** `the_autograd_compiler` pointer and the Python object's reference count.
- **Writer(s):** `set_autograd_compiler()` does `Py_INCREF(obj)` and stores the new pointer, returning the old pointer (which the caller is expected to decref).
- **Reader(s):** Any concurrent access to `the_autograd_compiler`.
- **Race scenario:** Even with the GIL (in CPython 3.13-), the sequence `prior_compiler = the_autograd_compiler; ... Py_INCREF(obj); the_autograd_compiler = obj;` is not atomic as a whole. Under free-threading, Thread A could read `the_autograd_compiler` in `Engine::execute` (loads the function pointer, which was set via `Engine::set_compiled_autograd`) while Thread B is in the middle of `set_autograd_compiler`. Since `the_compiled_autograd` in engine.cpp is `std::atomic`, the function pointer itself is safe, but the `PyObject*` `the_autograd_compiler` is not atomic and its reference count manipulation is not synchronized.
- **Suggested fix:** Use a mutex or `std::atomic<PyObject*>` with proper refcount management.

### G2. `default_dyn_type_int` non-atomic read/write

- **Shared state:** `default_dyn_type_int` (file-static `int`).
- **Writer(s):** `set_autograd_compiler()` writes it.
- **Reader(s):** `get_default_dyn_type()` reads it during `_compiled_autograd_impl`.
- **Race scenario:** Thread A is in `_compiled_autograd_impl` calling `get_default_dyn_type()` while Thread B calls `set_autograd_compiler`. The read/write is on a plain `int` with no synchronization. Under free-threading this is a data race (undefined behavior in C++). In practice this is a configuration flag so the consequence is merely using the wrong dynamic type, but it is technically UB.
- **Suggested fix:** Use `std::atomic<int>` or protect with the mutex.

### G3. `CacheNode` tree mutations during `check_dynamic_sizes` are not thread-safe

- **Shared state:** `CacheNode::expected_sizes`, `CacheNode::runtime_wrapper`, `CacheNode::compiled_fn`.
- **Writer(s):** `check_dynamic_sizes()` modifies `expected_sizes` and can null out `runtime_wrapper`/`compiled_fn` on a dynamic-shape cache miss.
- **Reader(s):** A concurrent `is_cache_empty()` call reads the tree. More importantly, if the mutex were ever relaxed to allow concurrent backwards (a stated goal), two threads could be in `check_dynamic_sizes` on the same `CacheNode` simultaneously.
- **Race scenario:** Currently the mutex serializes backward calls, so this is not exploitable. However, `is_cache_empty()` and `clear_cache()` have no synchronization. If Thread A is in `check_dynamic_sizes` modifying `expected_sizes` and Thread B calls `clear_cache`, Thread B destroys the node. Even `is_cache_empty` reads `next.empty()` and `compiled_fn` without synchronization with a concurrent backward.
- **Suggested fix:** `clear_cache` and `is_cache_empty` should acquire the compiled autograd mutex.

### G4. Static local variables in `call_begin_capture` and `_compiled_autograd_impl`

- **Shared state:** Several `static PyObject*` locals created via `PyUnicode_InternFromString` (e.g., `method_name` in `call_begin_capture`, `call_end_capture`). Also the `static bool flag` lambda in `_compiled_autograd_impl` line 1077.
- **Writer(s):** First thread to reach each initialization.
- **Reader(s):** All subsequent threads.
- **Race scenario:** C++ guarantees thread-safe initialization of function-local statics (magic statics). However, the `static bool flag` lambda at line 1077 is guarded by the C++ runtime's static initialization lock, but the lambda body calls `bind_function` which calls into Python. If another thread is also blocked waiting on this static initialization, and the Python call triggers code that tries to enter the same static initialization, it could deadlock. This is unlikely but worth noting. The `PyUnicode_InternFromString` calls are safe as long as they're only called with the GIL held.
- **Suggested fix:** Move one-time initialization to module init time rather than using lazy static locals.

## Minor Issues

### M1. `Py_MOD_GIL_NOT_USED` is overly broad

- **Shared state:** All module-level functions.
- **Writer(s):** N/A.
- **Reader(s):** Python runtime.
- **Race scenario:** By declaring `Py_MOD_GIL_NOT_USED`, all five exported C methods (`set_autograd_compiler`, `clear_cache`, `is_cache_empty`, `set_verbose_logger`, `call_cpp_tensor_pre_hooks`) can be called without the GIL on 3.14t. None of these functions have internal synchronization adequate for concurrent calls. This is the root enabler for S1-S4.
- **Suggested fix:** Remove `Py_MOD_GIL_NOT_USED` until proper internal synchronization is added. This is the single most impactful fix.

### M2. `CompiledAutogradThreadingDebugCheck` is a best-effort check, not enforcement

- **Shared state:** `num_threads_in_compiled_autograd` (atomic int32 in engine.cpp).
- **Writer(s):** Incremented/decremented by `CompiledAutogradThreadingDebugCheck` in `Engine::execute`.
- **Reader(s):** Checked in `Engine::execute` before entering compiled autograd.
- **Race scenario:** The check `num_threads_in_compiled_autograd.load() == 0` is a TOCTOU race: two threads could both see 0, both proceed, and both try to acquire the compiled autograd mutex. The mutex protects correctness, but the check's error message says "Re-entrant into Compiled Autograd ... is not yet supported" -- this is misleading because the mutex (via `try_lock`) will catch the actual conflict. The `_thread_check.release()` call before entering `compiled_autograd` means the counter is decremented before the mutex is acquired, so a third thread could slip through the check.
- **Suggested fix:** This is informational only. The real protection is the mutex. Consider removing the debug check or making it accurate.

### M3. THPObjectPtr release() in CacheNode destructor leaks on shutdown

- **Shared state:** `CacheNode::runtime_wrapper`, `CacheNode::compiled_fn`.
- **Writer(s):** `CacheNode::~CacheNode()`.
- **Reader(s):** N/A.
- **Race scenario:** Not a race per se, but `~CacheNode()` checks `Py_IsInitialized()` and leaks if the interpreter is finalizing. Under free-threading, interpreter finalization and CacheNode destruction could interleave in surprising ways. If a thread is still executing compiled autograd while the interpreter starts finalizing, `Py_IsInitialized` could return true initially but become false by the time `Py_DECREF` runs inside `THPObjectPtr::free()`.
- **Suggested fix:** Register an atexit handler to clear the cache before interpreter finalization, or ensure the static `CacheNode` root is destroyed at the right phase of shutdown.
