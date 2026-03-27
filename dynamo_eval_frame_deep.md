# Deep Free-Threading Audit: dynamo eval frame path

## Architecture Summary

The eval frame interception path works as follows:

1. **Eval frame shim** (`eval_frame.c`): When dynamo is active, `_PyInterpreterState_SetEvalFrameFunc` installs `dynamo_custom_eval_frame_shim` as the interpreter-wide frame evaluator. This fires on every Python function call across all threads.

2. **Per-thread callback** (`eval_frame.c`): A thread-specific storage key (`eval_frame_callback_key`) holds the per-thread dynamo callback (None = disabled, False = run-only, callable = compile). The shim reads this to decide what to do.

3. **ExtraState** (`extra_state.cpp/h`): Attached to **code objects** via `_PyCode_SetExtra`/`_PyCode_GetExtra`. Code objects are shared across all threads executing the same function. ExtraState holds:
   - `cache_entry_list`: a `std::list<CacheEntry>` of compiled code variants
   - `precompile_entries`: a `std::list<PrecompileEntry>` of precompiled entries
   - `frame_state`: a `py::dict` for tracking dynamic shapes
   - `strategy`: a `FrameExecStrategy` controlling skip/run-only behavior

4. **Cache lookup** (`extra_state.cpp:lookup`): Iterates `precompile_entries` then `cache_entry_list`, calling `run_root_guard_manager` on each entry's guard. On hit, optionally calls `move_to_front` (LRU). On miss, returns `Py_None` to trigger compilation.

5. **Cache insertion** (`extra_state.cpp:create_cache_entry`): After compilation, `emplace_front`/`emplace_back` adds a new `CacheEntry` to the list.

6. **Compilation** (`eval_frame_cpp.cpp:dynamo__custom_eval_frame`): On cache miss, calls the Python-side callback (`dynamo_call_callback`) which invokes `torch._dynamo.convert_frame`, then inserts the result into the cache.

**Critical concurrency property**: Under free-threading, two threads can call the same Python function simultaneously, meaning they share the same code object and thus the same ExtraState. All operations on ExtraState (lookup, insertion, move_to_front, invalidation, strategy mutation) happen without any synchronization.

---

## SEVERE Issues

### S1: Concurrent cache lookup and insertion tear the cache entry list

- **Shared state:** `ExtraState::cache_entry_list` (`std::list<CacheEntry>`) attached to a code object shared by all threads.
- **Writer(s):** Any thread that gets a cache miss in `dynamo__custom_eval_frame` calls `create_cache_entry` (line 660, `eval_frame_cpp.cpp`), which calls `emplace_front`/`emplace_back` on the list (lines 213-216, `extra_state.cpp`).
- **Reader(s):** Any thread calling the same function enters `lookup` (line 536-543, `eval_frame_cpp.cpp`) which iterates the list with a range-for loop (line 157, `extra_state.cpp`).
- **Race scenario:** Thread A is iterating `cache_entry_list` in `lookup`, currently at some iterator position. Thread B completes compilation and calls `create_cache_entry`, which does `emplace_front`. In `std::list`, `emplace_front` modifies the list's internal sentinel/head node pointers. While `std::list` iterators are not invalidated by insertion, the internal linked-list node manipulation is not atomic. If Thread B's `emplace_front` is modifying the `next` pointer of the sentinel node while Thread A is comparing its iterator to `end()` (which points to the sentinel), Thread A may see a partially written pointer, leading to undefined behavior, crashes, or infinite loops. Even if individual node pointers happen to be written atomically at the hardware level, `std::list::emplace_front` typically modifies multiple pointers (new node's prev/next, head's prev, sentinel's next) non-atomically, creating a window where the list is structurally inconsistent.
- **Suggested fix:** Protect ExtraState access with a mutex (or reader-writer lock). Alternatively, use an RCU-like scheme: build a new list snapshot on insertion and atomically swap a pointer, letting old readers finish on the old snapshot.

### S2: Concurrent move_to_front during cache lookup causes list corruption

- **Shared state:** `ExtraState::cache_entry_list` (same list as S1).
- **Writer(s):** `lookup` itself calls `extra_state->move_to_front(found)` (line 198, `extra_state.cpp`) after a cache hit. This calls `std::list::splice` (line 36-39, `extra_state.cpp`) which re-links nodes within the list.
- **Reader(s):** Another thread concurrently in `lookup` iterating the same list.
- **Race scenario:** Thread A and Thread B both call the same function. Both enter `lookup`. Thread A finds entry X valid, Thread B finds entry Y valid. Both call `move_to_front`. `splice` modifies up to 6 pointers (detach node from old position, reattach at front). Two concurrent `splice` calls on the same list is undefined behavior -- they can corrupt the doubly-linked list structure, causing nodes to be orphaned, iterators to loop infinitely, or segfaults when following dangling pointers.
- **Suggested fix:** Same as S1 -- lock the ExtraState. Or, since LRU reordering is an optimization, consider making it a no-op under free-threading if full locking is too expensive.

### S3: Double initialization of ExtraState (init_and_set_extra_state race)

- **Shared state:** The code object's extra data slot (accessed via `_PyCode_GetExtra`/`_PyCode_SetExtra`).
- **Writer(s):** `dynamo__custom_eval_frame` lines 504-506: checks `extra == nullptr`, then calls `init_and_set_extra_state`. Also `dynamo_set_code_exec_strategy` (lines 688-690) and `_load_precompile_entry` (lines 276-278) do the same check-then-init pattern.
- **Reader(s):** Any thread entering the same function.
- **Race scenario:** Thread A and Thread B both call the same (never-before-seen) function. Both call `get_extra_state` and get `nullptr`. Both enter the `if (extra == nullptr)` branch. Both call `init_and_set_extra_state`, which asserts `get_extra_state(code) == nullptr` (line 117, `extra_state.cpp`). Thread A sets the extra state. Thread B's CHECK fails, crashing the process. Even if the CHECK were removed, Thread B's `_PyCode_SetExtra` would overwrite Thread A's pointer, leaking Thread A's ExtraState and any cache entries Thread A may have already added to it. Note: while `_PyCode_SetExtra` in CPython 3.14t uses internal critical sections for the pointer swap itself, the get-then-check-then-set sequence across two separate API calls is not atomic.
- **Suggested fix:** Use a compare-and-swap pattern or lock around the init_and_set_extra_state call. One approach: attempt to set the extra, and if it was already set by another thread, delete the new one and use the existing one. Alternatively, use a global lock keyed on the code object pointer for the initialization path.

### S4: Concurrent compilation produces duplicate cache entries and corrupted frame_state

- **Shared state:** `ExtraState::cache_entry_list` and `ExtraState::frame_state`.
- **Writer(s):** After a cache miss, `dynamo__custom_eval_frame` calls the Python callback for compilation (line 614) and then `create_cache_entry` (line 660). `frame_state` is passed to the callback (line 615) which mutates it during compilation to track dynamic shapes.
- **Reader(s):** Other threads in `lookup` reading the cache list; the callback itself reading/writing `frame_state`.
- **Race scenario:** Thread A and Thread B both call the same function with the same inputs. Both get cache misses (because neither has compiled yet). Both invoke the Python compilation callback concurrently. Both mutate `frame_state` (a `py::dict`) concurrently -- dict operations are not thread-safe under free-threading without the GIL. Both complete compilation and both call `create_cache_entry`, adding duplicate entries to the list. The duplicate entries waste memory and evaluation time, but the `frame_state` corruption is the real danger: inconsistent dynamic shape tracking can lead to incorrect guards or incorrect compiled code.
- **Suggested fix:** Use a per-ExtraState lock that is held during compilation (or at minimum during frame_state mutation). Consider a "compilation in progress" flag so only one thread compiles for a given code object while others wait or fall back to the interpreter.

---

## Significant Issues

### G1: enable_eval_frame_shim / enable_eval_frame_default race on previous_eval_frame

- **Shared state:** The file-static `previous_eval_frame` function pointer (line 187-190, `eval_frame.c`) and the interpreter-level eval frame function.
- **Writer(s):** `enable_eval_frame_shim` (called from `increment_working_threads`) and `enable_eval_frame_default` (called from `decrement_working_threads`). These are called from `set_eval_frame` when threads enable/disable dynamo.
- **Reader(s):** `dynamo_eval_frame_default` reads `previous_eval_frame` on every default frame evaluation (line 206).
- **Race scenario:** Thread A calls `set_eval_frame(callback)` which calls `increment_working_threads` -> `enable_eval_frame_shim`, which saves the current eval frame func to `previous_eval_frame` and installs the shim. Concurrently, Thread B calls `set_eval_frame(Py_None)` which calls `decrement_working_threads` -> `enable_eval_frame_default`, which restores `previous_eval_frame` and sets it to NULL. Now Thread A (or Thread C) is inside `dynamo_eval_frame_default` and reads `previous_eval_frame` as NULL (line 206), falling through to `_PyEval_EvalFrameDefault` which may be wrong if there was a real previous eval frame. Worse, the non-atomic read of the function pointer during a concurrent write is undefined behavior.
- **Suggested fix:** Use an atomic for `previous_eval_frame`, or protect the enable/disable path with a mutex. Also note that `active_dynamo_threads` in `ModuleState` is incremented/decremented non-atomically (lines 552, 568).

### G2: active_dynamo_threads counter is not atomic

- **Shared state:** `ModuleState::active_dynamo_threads` (line 25, `eval_frame.c`).
- **Writer(s):** `increment_working_threads` (line 552) and `decrement_working_threads` (line 568), called from `set_eval_frame` on any thread.
- **Reader(s):** Same functions check the value to decide whether to install/uninstall the eval frame shim.
- **Race scenario:** Thread A and Thread B both call `set_eval_frame` with a non-None callback simultaneously. Both read `active_dynamo_threads == 0`. Both increment it to 1 (instead of 2). Later, when Thread A disables dynamo and decrements to 0, the shim is uninstalled even though Thread B is still active. Thread B's eval frame callback is now silently ignored, and compiled code stops being used.
- **Suggested fix:** Use `std::atomic<int>` or the C `_Py_atomic_*` primitives. The install/uninstall of the eval frame shim should be done under a lock since it involves multiple state changes.

### G3: ExtraState::invalidate concurrent with lookup causes use-after-free

- **Shared state:** `CacheEntry` fields (`code`, `root_mgr`, `guard_manager`, `backend`) within `ExtraState::cache_entry_list`.
- **Writer(s):** `ExtraState::invalidate` (line 52-71, `extra_state.cpp`) sets `cache_entry->code = py::none()`, `root_mgr = nullptr`, `backend = py::none()`, then calls `move_to_back`.
- **Reader(s):** `lookup` iterates entries and reads `cache_entry.backend`, `cache_entry.root_mgr`, etc.
- **Race scenario:** Thread A is in `lookup`, has obtained a reference to a CacheEntry, and is about to call `run_root_guard_manager(cache_entry.root_mgr, ...)`. Thread B calls `invalidate` on that same entry, setting `root_mgr = nullptr`. Thread A then calls `run_root_guard_manager` with a nullptr, which returns false (safe), OR with a partially-torn pointer (unsafe). More critically, if Thread A reads `cache_entry.code` after `invalidate` has set it to `py::none()`, it would try to execute `Py_None` as a code object. The `invalidate` function also calls `move_to_back` which does `splice`, conflicting with Thread A's ongoing iteration.
- **Suggested fix:** Lock ExtraState during invalidation. Or use an atomic flag on CacheEntry that lookup checks after guard evaluation to detect concurrent invalidation.

### G4: PreserveGlobalState saves/restores a process-global random state across threads

- **Shared state:** Python's `random` module global state.
- **Writer(s):** `PreserveGlobalState` constructor calls `random.getstate()` and destructor calls `random.setstate()` (lines 233-254, `eval_frame_cpp.cpp`).
- **Reader(s):** Any Python code using the `random` module in any thread.
- **Race scenario:** Thread A enters compilation, saves the random state as S1. Thread B is using `random.randint()` and advances the state to S2. Thread A finishes compilation, restores state to S1, silently reverting Thread B's state changes. Under free-threading, `random.getstate()`/`setstate()` on the global Random instance are not atomic operations either, so the state could be corrupted if Thread B is calling random functions concurrently with the save/restore.
- **Suggested fix:** Remove `PreserveGlobalState` for free-threading builds, or use a per-thread random state. Alternatively, use `random.Random()` instances rather than the module-level functions.

### G5: convert_frame_get_fail_callback static local is not thread-safe

- **Shared state:** `static std::optional<py::object> convert_frame_get_fail_callback` inside the `eval_custom` lambda in `dynamo__custom_eval_frame` (line 425).
- **Writer(s):** First thread to hit the `EvalFrameOverride::ERROR` path initializes it (lines 445-451). The `atexit` handler resets it to `std::nullopt`.
- **Reader(s):** Every thread in `eval_custom` that hits `EvalFrameOverride::ERROR` reads and calls it (line 454).
- **Race scenario:** Thread A and Thread B both enter `eval_custom` with `EvalFrameOverride::ERROR`. Both see `!convert_frame_get_fail_callback` as true. Both import the module and create the callback. Thread B's write overwrites Thread A's, leaking the py::object. Additionally, reading the `std::optional` while another thread is writing to it is a data race (UB). The `atexit` handler resetting it to `std::nullopt` while another thread is reading it is also a data race.
- **Suggested fix:** Use `std::call_once` or `py::gil_safe_call_once_and_store` for initialization. For the atexit handler, use an atomic flag or accept the leak at shutdown.

---

## Minor Issues

### M1: random_module lazy initialization is racy

- **Shared state:** `static PyObject* random_module` (line 220, `eval_frame_cpp.cpp`).
- **Writer(s):** `get_random_module()` lazily initializes it on first call (lines 221-225).
- **Reader(s):** Every compilation path via `PreserveGlobalState`.
- **Race scenario:** Two threads both see `random_module == nullptr` and both import the module. One import result is leaked. The pointer write itself is likely atomic on most architectures, so the worst case is a leaked reference -- the module object itself is safe because Python module imports are protected internally.
- **Suggested fix:** Use `std::call_once` or `py::gil_safe_call_once_and_store` for the lazy initialization.

### M2: breakpoint_code_objects unordered_set accessed without synchronization

- **Shared state:** `static std::unordered_set<PyCodeObject*> breakpoint_code_objects` (line 27, `eval_frame_cpp.cpp`).
- **Writer(s):** `register_breakpoint_code` inserts into the set (line 75).
- **Reader(s):** `eval_custom` lambda calls `breakpoint_code_objects.count()` (line 463).
- **Race scenario:** A thread registers a breakpoint code object (typically from a debugger/UI thread) while another thread is iterating or probing the hash set in `eval_custom`. `std::unordered_set::insert` can trigger rehashing, which invalidates all iterators and moves all elements. A concurrent `count()` during rehashing reads freed or moved memory.
- **Suggested fix:** Use a mutex around accesses, or a concurrent set. Since this is a debugging feature, a simple mutex is fine.

### M3: guard_error_hook and guard_complete_hook are read/written without atomics

- **Shared state:** `PyObject* guard_error_hook` and `PyObject* guard_complete_hook` (lines 21-22, `eval_frame.c`).
- **Writer(s):** `set_guard_error_hook` and `set_guard_complete_hook` from Python.
- **Reader(s):** `lookup` reads `guard_error_hook` (line 173, `extra_state.cpp`). `dynamo__custom_eval_frame` reads `guard_complete_hook` (line 564).
- **Race scenario:** A thread sets `guard_complete_hook = Py_XNewRef(obj)` while another thread reads the old pointer value and calls it. The old object may have been decref'd to zero by `Py_XSETREF` in `set_guard_error_hook`, causing a use-after-free. For `guard_complete_hook`, the `set_guard_complete_hook` function doesn't use `Py_XSETREF` and manually swaps, but the read in `dynamo__custom_eval_frame` is still a non-atomic load racing with the store.
- **Suggested fix:** Use `_Py_atomic_load_ptr`/`_Py_atomic_store_ptr` for the pointer reads/writes, and ensure reference counting is correct across the race window.

### M4: bytecode_debugger_callback_obj read/write race

- **Shared state:** `static PyObject* bytecode_debugger_callback_obj` (line 25, `eval_frame_cpp.cpp`).
- **Writer(s):** `set_bytecode_debugger_callback` (line 58) uses `Py_XSETREF`.
- **Reader(s):** `get_bytecode_debugger_callback` (line 66) and `eval_custom` (line 463) read the pointer.
- **Race scenario:** Similar to M3 -- a thread reads the pointer while another thread is setting it via `Py_XSETREF`, which decrefs and then stores. The reading thread may use a pointer that has been decremented to zero.
- **Suggested fix:** Use atomic pointer operations.

### M5: Extra state strategy field has benign TOCTOU but can cause stale reads

- **Shared state:** `ExtraState::strategy` (a `FrameExecStrategy` struct with two `FrameAction` enum fields).
- **Writer(s):** `extra_state_set_exec_strategy` called from `dynamo__custom_eval_frame` (line 649) and `dynamo_set_code_exec_strategy` / `dynamo_skip_code_recursive`.
- **Reader(s):** `extra_state_get_exec_strategy` called early in `dynamo__custom_eval_frame` (line 509).
- **Race scenario:** Thread A reads strategy as DEFAULT and proceeds to compile. Thread B concurrently sets strategy to SKIP (e.g., because the function should be skipped). Thread A doesn't see the update and compiles unnecessarily. The struct copy is not atomic (two fields), so Thread A could theoretically read `cur_action=SKIP` but `recursive_action=DEFAULT` during a torn read. In practice, the consequence is wasted compilation or one frame not being skipped, which is annoying but not a correctness/safety issue.
- **Suggested fix:** Use `std::atomic<uint64_t>` packing both fields, or just accept the benign race. The real fix comes from locking ExtraState as recommended in the SEVERE issues.
