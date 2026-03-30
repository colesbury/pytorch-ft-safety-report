# Free-Threading Audit: Dynamo Eval Frame Interception Path

## Architecture Summary

The dynamo eval frame path intercepts every Python function call via PEP 523.
The key data flow is:

1. CPython calls `dynamo_custom_eval_frame_shim` for every frame evaluation.
2. The shim reads a **thread-local** callback (`eval_frame_callback_key` via
   `Py_tss_t`) to decide: `None` (skip), `Py_False` (run-only), or a callable
   (full dynamo).
3. `dynamo__custom_eval_frame` retrieves `ExtraState` from the code object via
   `_PyCode_GetExtra`. Code objects are shared across all threads calling the
   same function, so `ExtraState` is shared.
4. `ExtraState` holds: `cache_entry_list` (`std::list<CacheEntry>`),
   `precompile_entries` (`std::list<PrecompileEntry>`), `frame_state`
   (`py::dict`), and `strategy` (`FrameExecStrategy`).
5. `lookup()` iterates `cache_entry_list` evaluating guards. On hit,
   `move_to_front()` splices the entry for LRU. On miss, the Python callback
   compiles, and `create_cache_entry()` inserts into the list.
6. `ExtraState::invalidate()` is called from Python (via dict watcher
   callbacks / mutation tracking) on whatever thread triggers the mutation.

The callback is per-thread (TSS), but ExtraState, cache entries, and several
global variables are shared across threads.

## SEVERE Issues

### 1. Concurrent access to `cache_entry_list` (std::list) during lookup, insertion, invalidation, and LRU reordering

- **Shared state:** `ExtraState::cache_entry_list` -- a `std::list<CacheEntry>`
  attached to a code object shared across all threads.
- **Writer(s):**
  - `create_cache_entry()` -- called after a cache miss during compilation
    (`emplace_front`/`emplace_back`, plus iterator bookkeeping).
  - `ExtraState::move_to_front()` / `move_to_back()` -- called from `lookup()`
    on cache hit and from `invalidate()`.
  - `ExtraState::invalidate()` -- called from Python-side dict watcher /
    mutation guard callbacks on **whatever thread triggers the mutation**
    (e.g., a data-loader thread doing a lazy import that triggers a dict
    watcher on a guarded `__dict__`). This calls `cache_entry->invalidate()`
    which mutates the `CacheEntry` fields, then calls `move_to_back()` which
    splices the list.
  - `destroy_extra_state()` -- the `_PyCode_SetExtra` free function, called
    when a code object is deallocated or when `reset_code` is called.
- **Reader(s):**
  - `lookup()` -- iterates `cache_entry_list` with a range-for loop,
    reading each `CacheEntry`'s fields (`root_mgr`, `backend`, `code`,
    `diff_guard_root_mgr`).
  - `extract_cache_entry()` -- returns `cache_entry_list.front()`.
  - `_debug_get_cache_entry_list()` -- iterates the list from Python.
  - `CacheEntry::next()` -- walks the list via `_owner_loc` iterator.
- **Race scenario:** Thread A calls a compiled function and is inside
  `lookup()`, iterating `cache_entry_list` and evaluating guards on each
  `CacheEntry`. Thread B (a data-loader thread, or another compile thread)
  triggers a dict watcher callback that calls `ExtraState::invalidate()`,
  which mutates a `CacheEntry`'s fields (setting `code` to `None`,
  `root_mgr` to `nullptr`) and splices the list via `move_to_back()`.
  Thread A is now iterating a structurally modified `std::list` -- the
  iterator may be invalidated or point to moved nodes, and the `CacheEntry`
  fields Thread A is reading (`root_mgr`, `code`) are being concurrently
  overwritten. This is undefined behavior on the `std::list` internals and
  on the `CacheEntry` fields. Likely crash: null dereference of `root_mgr`
  in `run_root_guard_manager`, or use of a dangling iterator.
- **Tier:** **Tier 1** for the invalidation path (dict watcher callbacks fire
  on data-loader threads). **Tier 2** for concurrent lookup + create.
- **Suggested fix:** Protect `ExtraState` with a mutex. The critical sections
  are: `lookup()` (read lock), `create_cache_entry()` (write lock),
  `move_to_front()`/`move_to_back()` (write lock), `invalidate()` (write
  lock). A `std::shared_mutex` (reader-writer lock) on `ExtraState` would
  allow concurrent lookups while serializing mutations. Alternatively, use
  atomic operations on the list structure (e.g., RCU-style read-copy-update).

### 2. Concurrent access to `precompile_entries` (std::list)

- **Shared state:** `ExtraState::precompile_entries` -- a
  `std::list<PrecompileEntry>`.
- **Writer(s):**
  - `_load_precompile_entry()` -- called from Python to add entries.
  - `_reset_precompile_entries()` -- called from Python to clear entries.
- **Reader(s):**
  - `lookup()` -- iterates `precompile_entries` in its range-for loop,
    calling `run_root_guard_manager` on each entry.
- **Race scenario:** Thread A is inside `lookup()` iterating
  `precompile_entries`. Thread B calls `_reset_precompile_entries()` or
  `_load_precompile_entry()`, structurally modifying the list. Thread A's
  iterator is invalidated. Crash from corrupted list internals.
- **Tier:** **Tier 2** (requires concurrent compiled function execution and
  precompile entry manipulation).
- **Suggested fix:** Same mutex on `ExtraState` as issue #1.

### 3. `init_and_set_extra_state` check-then-act TOCTOU

- **Shared state:** The "extra" slot on the code object (accessed via
  `_PyCode_GetExtra` / `_PyCode_SetExtra`).
- **Writer(s):** `init_and_set_extra_state()` -- checks
  `get_extra_state(code) == nullptr`, then allocates and sets.
  Called from: `dynamo__custom_eval_frame` (line 505-506),
  `dynamo_set_code_exec_strategy` (line 689-690),
  `dynamo_skip_code_recursive` (line 702-703),
  `_load_precompile_entry` (line 277-278).
- **Reader(s):** `get_extra_state()` in every caller.
- **Race scenario:** Thread A and Thread B both call the same function for
  the first time. Both see `extra == nullptr`. Both call
  `init_and_set_extra_state()`. The CHECK inside will fire on the second
  thread (since the first already set it), causing an assertion failure /
  abort. Even if the CHECK is removed, the second `_PyCode_SetExtra` call
  will overwrite the first's pointer, and `destroy_extra_state` will be
  called on the first allocation, leading to use-after-free if Thread A
  already has a pointer to the first ExtraState.
- **Tier:** **Tier 2** (requires two threads hitting the same un-compiled
  function simultaneously).
- **Suggested fix:** Use a compare-and-swap or a global lock around
  init_and_set_extra_state. Alternatively, use `_PyCode_GetExtra` +
  `_PyCode_SetExtra` atomically (CPython 3.14t may need internal locking on
  code extras for this). A simple approach: use `std::call_once` or a
  per-code-object lock.

### 4. `breakpoint_code_objects` (std::unordered_set) concurrent access

- **Shared state:** `breakpoint_code_objects` -- a file-scope
  `std::unordered_set<PyCodeObject*>`.
- **Writer(s):** `register_breakpoint_code()` -- called from Python to add
  code objects.
- **Reader(s):** `dynamo__custom_eval_frame` at line 463 -- calls
  `breakpoint_code_objects.count(cached_code)` during eval_custom.
- **Race scenario:** Thread A is evaluating a frame in `eval_custom` and
  calls `breakpoint_code_objects.count()`. Thread B simultaneously calls
  `register_breakpoint_code()` which calls `insert()` on the same
  `unordered_set`. Concurrent read + write on `std::unordered_set` is
  undefined behavior (potential rehash during read).
- **Tier:** **Tier 2** (requires concurrent compiled function execution +
  breakpoint registration, though breakpoint registration is relatively rare).
- **Suggested fix:** Use a `std::mutex` around accesses, or use a concurrent
  set. Since this is a debugging feature, a simple mutex is appropriate.

## Significant Issues

### 5. `ExtraState::strategy` non-atomic read/write

- **Shared state:** `ExtraState::strategy` -- a `FrameExecStrategy` struct
  containing two `FrameAction` enum values.
- **Writer(s):** `extra_state_set_exec_strategy()` -- called from
  `dynamo__custom_eval_frame` (line 649) after compilation, and from
  `dynamo_set_code_exec_strategy` / `dynamo_skip_code_recursive`.
- **Reader(s):** `extra_state_get_exec_strategy()` -- called from
  `dynamo__custom_eval_frame` (line 509) at the start of every frame
  evaluation.
- **Race scenario:** Thread A is evaluating a frame and reads `strategy`.
  Thread B just finished compiling the same function and writes a new
  `strategy` (e.g., setting `cur_action = SKIP`). The struct write is not
  atomic (it's two enum fields), so Thread A could read a torn value where
  `cur_action` is updated but `recursive_action` is stale, or vice versa.
  This could cause incorrect frame action decisions (e.g., skipping a frame
  that should be compiled, or vice versa).
- **Tier:** **Tier 2**.
- **Suggested fix:** Protect with the same `ExtraState` mutex from issue #1,
  or pack the two enum values into a single `std::atomic<uint32_t>`.

### 6. `ExtraState::frame_state` (py::dict) concurrent access

- **Shared state:** `ExtraState::frame_state` -- a `py::dict` (Python dict).
- **Writer(s):** The Python-side compilation callback receives a pointer to
  `frame_state` (via `extract_frame_state`) and mutates it during
  compilation to record dynamic shape information.
- **Reader(s):** Another thread compiling the same function, or the main
  thread accessing frame_state while a recompilation occurs on another thread.
- **Race scenario:** Thread A and Thread B both trigger compilation for the
  same code object. Both receive the same `frame_state` dict pointer. Both
  mutate it concurrently from Python. Under free-threading, concurrent
  mutations to the same Python dict without holding the per-object lock can
  cause corruption (Python 3.14t has per-object locks for dicts, but only
  for Python-level access via bytecode -- C-level access via `PyDict_SetItem`
  from multiple threads without the GIL is still racy unless using the
  critical section API).
- **Tier:** **Tier 2**.
- **Suggested fix:** Protect compilation with a per-code-object lock, or
  ensure `frame_state` access goes through Python critical sections.

### 7. `eval_frame_override` global variable non-atomic access

- **Shared state:** `eval_frame_override` -- a file-scope
  `EvalFrameOverride` enum (in `eval_frame_cpp.cpp`, line 342).
- **Writer(s):** `set_eval_frame_override()` -- called from Python.
- **Reader(s):** `dynamo__custom_eval_frame` at line 443 -- reads
  `eval_frame_override` inside `eval_custom` lambda on every compiled frame
  evaluation.
- **Race scenario:** Thread A is evaluating frames and reading
  `eval_frame_override`. Thread B calls `set_eval_frame_override()` to
  change it. The read is not atomic. On x86 this is likely benign (enum is
  int-sized, loads/stores are naturally atomic), but it's technically UB in
  C++ and could be problematic on weaker architectures.
- **Tier:** **Tier 2**.
- **Suggested fix:** Make `eval_frame_override` a `std::atomic<EvalFrameOverride>`.

### 8. `guard_error_hook` and `guard_complete_hook` global PyObject* pointers

- **Shared state:** `guard_error_hook` and `guard_complete_hook` -- global
  `PyObject*` pointers (declared in `eval_frame.c`).
- **Writer(s):**
  - `set_guard_error_hook()` -- uses `Py_XSETREF` to swap the pointer.
  - `set_guard_complete_hook()` -- manually swaps the pointer.
- **Reader(s):**
  - `lookup()` in `extra_state.cpp` reads `guard_error_hook` (line 173).
  - `dynamo__custom_eval_frame` reads `guard_complete_hook` (line 564).
- **Race scenario:** Thread A is in `lookup()` and reads `guard_error_hook`
  as non-null, then enters the error handling path using the pointer.
  Thread B calls `set_guard_error_hook(None)`, which `Py_XSETREF`s the
  pointer to NULL and decrefs the old object. Thread A now uses a freed
  `PyObject*`. This is a use-after-free.
  Same pattern for `guard_complete_hook`.
- **Tier:** **Tier 2** (hooks are typically set once during setup, but
  nothing prevents them from being changed during execution).
- **Suggested fix:** Use `std::atomic<PyObject*>` and ensure the old object's
  refcount is managed with appropriate ordering. Or protect with a lock.

### 9. `previous_eval_frame` function pointer race in enable/disable

- **Shared state:** `previous_eval_frame` -- a file-scope function pointer
  (in `eval_frame.c`, line 187).
- **Writer(s):**
  - `enable_eval_frame_shim()` -- reads the current eval frame func, saves
    it in `previous_eval_frame`, and installs the dynamo shim.
  - `enable_eval_frame_default()` -- restores `previous_eval_frame` and sets
    it to NULL.
- **Reader(s):**
  - `dynamo_eval_frame_default()` -- reads `previous_eval_frame` to decide
    whether to call it or `_PyEval_EvalFrameDefault`.
- **Race scenario:** Thread A calls `set_eval_frame(callback)` which calls
  `increment_working_threads` -> `enable_eval_frame_shim`. Thread B
  simultaneously calls `set_eval_frame(None)` which calls
  `decrement_working_threads` -> `enable_eval_frame_default`. The
  read-modify-write on `previous_eval_frame` is not atomic.
  Additionally, `_PyInterpreterState_SetEvalFrameFunc` sets the eval frame
  function on the **interpreter** (shared by all threads), not per-thread.
  If Thread B sets it back to `previous_eval_frame` while Thread A is still
  using the shim on a different core, Thread A's in-flight frame evaluation
  could see inconsistent state.
- **Tier:** **Tier 2** (requires concurrent enable/disable of dynamo).
  Could also be **Tier 1** if the main thread enables dynamo while another
  thread is already executing frames.
- **Suggested fix:** Protect enable/disable with a mutex. The
  `active_dynamo_threads` counter is also not atomic and has the same
  TOCTOU issues.

### 10. `active_dynamo_threads` counter race

- **Shared state:** `ModuleState::active_dynamo_threads` -- an `int` in
  module state.
- **Writer(s):** `increment_working_threads()` and
  `decrement_working_threads()` -- read-increment/decrement-write without
  any synchronization.
- **Race scenario:** Thread A and Thread B both call
  `increment_working_threads`. Both read the same value (e.g., 0), both
  write 1. The counter is now 1 instead of 2. Later, Thread A calls
  `decrement_working_threads`, counter goes to 0, and
  `enable_eval_frame_default()` disables the shim while Thread B still
  expects it.
- **Tier:** **Tier 2**.
- **Suggested fix:** Use `std::atomic<int>` or protect with the same mutex
  as issue #9.

## Minor Issues

### 11. `is_skip_guard_eval_unsafe` non-atomic bool

- **Shared state:** `is_skip_guard_eval_unsafe` -- a file-scope `bool`
  (declared `extern` in `eval_frame.h`, defined in `eval_frame.c` line 728).
- **Writer(s):** `set_skip_guard_eval_unsafe()` -- from Python.
- **Reader(s):** `dynamo__custom_eval_frame` (line 543, 583) and `lookup()`
  (line 146 parameter) -- on every frame evaluation.
- **Race scenario:** Stale read. Thread A reads the old value while Thread B
  sets a new one. This could cause one extra frame to use the wrong guard
  evaluation mode, which is relatively benign (either an unnecessary full
  guard eval or one frame with skip-guard-eval when it shouldn't).
- **Tier:** **Tier 2**.
- **Suggested fix:** Make it `std::atomic<bool>`. Low priority since a stale
  read is benign.

### 12. `use_lru` non-atomic bool

- **Shared state:** `use_lru` -- a file-scope `bool` in `extra_state.cpp`
  (line 17).
- **Writer(s):** `_set_lru_cache()` -- from Python.
- **Reader(s):** `lookup()` (line 197) and `create_cache_entry()` (line 212)
  -- on every frame evaluation / cache creation.
- **Race scenario:** Stale read. Same as issue #11. A stale read might cause
  one lookup to skip or perform an unnecessary `move_to_front`, which is
  benign.
- **Tier:** **Tier 2**.
- **Suggested fix:** Make it `std::atomic<bool>`. Low priority.

### 13. `bytecode_debugger_callback_obj` and `random_module` global pointers

- **Shared state:** `bytecode_debugger_callback_obj` (line 25) and
  `random_module` (line 220) -- file-scope `PyObject*` pointers in
  `eval_frame_cpp.cpp`.
- **Writer(s):**
  - `set_bytecode_debugger_callback()` uses `Py_XSETREF` to swap.
  - `get_random_module()` lazily initializes `random_module`.
- **Reader(s):**
  - `get_bytecode_debugger_callback()` reads `bytecode_debugger_callback_obj`.
  - `PreserveGlobalState` constructor/destructor calls `get_random_module()`.
- **Race scenario:** For `bytecode_debugger_callback_obj`: similar to
  issue #8 but this is a debugging feature rarely used in production. For
  `random_module`: lazy init TOCTOU -- two threads could both see `nullptr`
  and both import, leaking one reference. The import itself is serialized by
  Python's import lock, so no crash, just a leaked reference.
- **Tier:** **Tier 2**.
- **Suggested fix:** For `random_module`, use C++11 `static` local for
  thread-safe lazy init. For `bytecode_debugger_callback_obj`, use
  `std::atomic<PyObject*>` with appropriate refcounting.

### 14. `convert_frame_get_fail_callback` static optional lazy init

- **Shared state:** `convert_frame_get_fail_callback` -- a `static
  std::optional<py::object>` inside the `eval_custom` lambda in
  `dynamo__custom_eval_frame` (line 425).
- **Writer(s):** First thread to reach the `if
  (!convert_frame_get_fail_callback)` branch initializes it.
- **Reader(s):** Subsequent threads read it.
- **Race scenario:** Two threads reach the check simultaneously. Both see
  `nullopt`. Both import and assign. The `std::optional` assignment is not
  atomic, so concurrent writes could corrupt the optional's internal state.
  However, this path requires `eval_frame_override == ERROR`, which is a
  niche configuration.
- **Tier:** **Tier 2**.
- **Suggested fix:** Use `std::call_once` or `static` local variable for
  thread-safe lazy initialization.

### 15. `c_recursion_limit` non-atomic int32_t

- **Shared state:** `c_recursion_limit` -- a file-scope `int32_t` (line 294
  of `eval_frame_cpp.cpp`).
- **Writer(s):** `dynamo_set_c_recursion_limit()`.
- **Reader(s):** `dynamo_get_c_recursion_limit()` and `CRecursionLimitRAII`.
- **Race scenario:** Stale read. Benign -- one frame might use a slightly
  outdated recursion limit.
- **Tier:** **Tier 2**.
- **Suggested fix:** `std::atomic<int32_t>`. Low priority.
