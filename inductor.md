# Free-Threading Audit: `torch/csrc/inductor/`

## Architecture Summary

The inductor C++ runtime (`torch/csrc/inductor/`) provides ahead-of-time (AOT)
compiled model execution infrastructure. The major subsystems:

1. **AOTI Runtime** (`aoti_runtime/`): `AOTInductorModelContainer` manages a pool
   of `AOTInductorModel` instances. Multiple threads call `run()` concurrently,
   each acquiring a model from the pool. Uses `models_mutex_` for pool management
   and `model_exec_mutex_` (shared_mutex) for inference vs. weight-swap
   exclusion. Constants are double-buffered (primary/secondary) with
   `use_secondary_` tracking which is active.

2. **AOTI Eager** (`aoti_eager/`): `AOTIPythonKernelHolder` is a dispatcher
   kernel functor — one instance per (op, dispatch key) pair, shared across all
   threads. Contains `aoti_kernel_cache_` (a `std::vector`) with **zero
   synchronization**.

3. **AOTI Runner/Package** (`aoti_runner/`, `aoti_package/`): Model loading,
   package extraction, pybind wrappers.

4. **AOTI Torch Shim** (`aoti_torch/`): Thin C wrappers around ATen ops.
   Essentially stateless — very clean.

5. **Static Launcher / cpp_prefix** (`static_launcher/`, `cpp_prefix.h`):
   Code included in all generated kernels, plus CUDA/XPU kernel launch helpers.

**Free-threading relevance:** Most of this code is pure C++ with its own locking
(or lack thereof). The GIL removal doesn't create new race conditions per se,
but it makes them far more likely to trigger: Python threads now run truly
concurrently, so the window for concurrent calls into these C++ APIs is wide
open.

## SEVERE Issues

### 1. `aoti_kernel_cache_` vector: concurrent read/write (use-after-free)

- **Shared state:** `AOTIPythonKernelHolder::aoti_kernel_cache_`
  (`std::vector<AOTIKernelMetadata>`) — `kernel_holder.h:77`
- **Writer:** `cache_miss()` at `kernel_holder.cpp:472` calls `push_back()`,
  which may reallocate the vector's internal buffer
- **Reader:** `cache_lookup()` at `kernel_holder.cpp:213-218` iterates the
  vector
- **Race scenario:** Thread A gets a cache miss and enters `cache_miss()`.
  Thread B concurrently calls the same op and enters `cache_lookup()`, iterating
  the vector. Thread A's `push_back` triggers reallocation — the buffer Thread B
  is iterating is freed. Thread B dereferences a dangling pointer.
- **Consequence:** Heap use-after-free, crash.
- **Tier:** 2 (multi-thread inference), but also 1 if data loader threads
  dispatch the same AOTI op.
- **Suggested fix:** Add `std::shared_mutex` to `AOTIPythonKernelHolder`.
  Shared lock in `cache_lookup`, exclusive lock in `cache_miss` with re-check
  before insert.

### 2. Concurrent `push_back` on `aoti_kernel_cache_`

- **Shared state:** Same vector as above
- **Writer(s):** Two concurrent `cache_miss()` calls both calling `push_back`
- **Race scenario:** Two threads both miss the cache for different input shapes,
  both compile kernels (long window — Python compilation), both call `push_back`
  concurrently. Two concurrent `push_back` on `std::vector` is UB — corrupted
  size/capacity, double-free.
- **Consequence:** Heap corruption, crash.
- **Tier:** 2.
- **Suggested fix:** Same mutex as issue 1.

### 3. `update_constant_buffer` races with concurrent `run()`

- **Shared state:** `constants_map_`, `constants_array_`, `constant_folded_`,
  `use_secondary_` in `AOTInductorModelContainer`
  (`model_container.h`)
- **Writer:** `update_constant_buffer()` calls `insert_or_assign` on the
  `ConstantMap` (an `unordered_map`) and writes `constant_folded_` — **without
  holding any lock**
- **Reader:** `run()` holds `model_exec_mutex_` in shared mode; models read
  `constants_map_` and `constants_array_` through `shared_ptr` copies
- **Race scenario:** Thread A calls `update_constant_buffer(use_inactive=false)`,
  modifying the active `ConstantMap`. Thread B is in `run()` with a shared lock,
  iterating the same map for constant lookup. Concurrent
  `insert_or_assign` + iteration on `unordered_map` is UB — bucket chain
  corruption.
- **Consequence:** Corrupted hash table internals, crash.
- **Suggested fix:** `update_constant_buffer()` must acquire `model_exec_mutex_`
  in unique mode when touching the active buffer (or at minimum, shared mode
  must exclude it).

### 4. `extract_constants_map` races with `update_constant_buffer`

- **Shared state:** `constants_map_`, `use_secondary_` in
  `AOTInductorModelContainer`
- **Writer:** `update_constant_buffer()` calls `insert_or_assign` on the map
- **Reader:** `extract_constants_map()` iterates the map via `find()` — no lock
  held
- **Race scenario:** Thread A iterates the map in `extract_constants_map`.
  Thread B concurrently inserts into the same map via `update_constant_buffer`.
  Rehash triggered by insert invalidates Thread A's iterator.
- **Consequence:** Iterator invalidation, crash.
- **Suggested fix:** Acquire at least shared lock on `model_exec_mutex_` in
  `extract_constants_map`.

### 5. `free_inactive_constant_buffer` races with `swap_constant_buffer`

- **Shared state:** `constant_blob_`, `constant_blob_secondary_`,
  `use_secondary_`
- **Writer:** `free_inactive_constant_buffer()` reads `use_secondary_` to select
  the inactive buffer, then resets it — **no lock held**
- **Writer:** `swap_constant_buffer()` flips `use_secondary_` under unique lock
- **Race scenario:** Thread A calls `free_inactive_constant_buffer()`, reads
  `use_secondary_ == false`, decides to free the secondary buffer. Thread B
  calls `swap_constant_buffer()`, flips `use_secondary_` to `true` (secondary
  is now active). Thread A frees the secondary buffer — which is now the active
  buffer being read by inference threads.
- **Consequence:** Use-after-free on constant tensor data during inference,
  crash.
- **Suggested fix:** `free_inactive_constant_buffer()` must hold
  `model_exec_mutex_` in unique mode.

### 6. `run_const_fold` (inactive buffer path) races with model state

- **Shared state:** Model instance's `constants_map_` and `constants_`
  (`shared_ptr` members on `AOTInductorModelBase`)
- **Writer:** `run_const_fold(inactive_buffer=true)` temporarily swaps a model's
  constant pointers to the inactive buffer, runs constant folding, then swaps
  back — holding `model_exec_mutex_` in **shared** mode only
- **Race scenario:** Thread A is in `run_const_fold`, swaps model's constants to
  inactive buffer. Thread B calls `swap_constant_buffer()` (unique lock), flips
  `use_secondary_`. Thread A's "swap back" reads `use_secondary_` (which has
  changed), and the model ends up pointing at the wrong buffer.
- **Consequence:** Model reads wrong constants, potential use-after-free if the
  buffer is subsequently freed.
- **Suggested fix:** Hold unique lock for the entirety of `run_const_fold`, or
  use a separate lock that `swap_constant_buffer()` also acquires.

### 7. `load_json_file` static JSON object

- **Shared state:** `static nlohmann::json json_obj` in `load_json_file()`
  (`model_package_loader.cpp:159`)
- **Writer:** Every call to `load_json_file()` overwrites this static object
- **Reader:** The caller holds a reference to the returned object
- **Race scenario:** Thread A calls `load_json_file("model_a.json")`, gets a
  reference to the static. Thread B calls `load_json_file("model_b.json")`,
  overwrites the static. Thread A's reference now points to model_b's data
  (or worse, a partially-overwritten JSON tree). `nlohmann::json` uses
  `std::map`/`std::vector` internally — concurrent read + overwrite is UB.
- **Consequence:** Crash or silent data corruption (wrong model config loaded).
  This is also a latent single-threaded bug.
- **Suggested fix:** Make the variable local and return by value.

### 8. `getAOTIModelRunnerRegistry()` — unsynchronized global `unordered_map`

- **Shared state:** Static `std::unordered_map<std::string, CreateAOTIModelRunnerFunc>`
  returned by `getAOTIModelRunnerRegistry()` (`model_container_runner.cpp:388`)
- **Writer:** `RegisterAOTIModelRunner` constructor (static initializer time)
- **Reader:** `AOTIPythonKernelHolder` constructor and `load_aoti_model_runner`
  call `find()`/`operator[]`
- **Race scenario:** Under free-threading, a lazy `import` on a background
  thread triggers a backend module's static initializer calling
  `RegisterAOTIModelRunner`, writing to the map. Main thread simultaneously
  reads the map to construct a kernel holder. Concurrent read + write on
  `unordered_map` is UB.
- **Tier:** 1 (lazy imports from data loader threads).
- **Suggested fix:** Wrap in `c10::Synchronized<>` or use `std::shared_mutex`.

## Significant Issues

### 9. `use_secondary_` read without synchronization in management APIs

- **Shared state:** `use_secondary_` (plain `bool`) in
  `AOTInductorModelContainer`
- **Writer:** `swap_constant_buffer()` flips it under unique lock
- **Reader:** `update_constant_buffer()`, `extract_constants_map()`,
  `get_constant_blob_ptr()`, `get_constants_map()`, `get_constants_array()`,
  `free_inactive_constant_buffer()` all read it **without any lock**
- **Race scenario:** Thread A calls `update_constant_buffer(use_inactive=true)`,
  reads `use_secondary_ == false`, decides to update secondary. Thread B calls
  `swap_constant_buffer()`, flips `use_secondary_` to `true`. Thread A's update
  goes to the **now-active** buffer instead of the inactive one.
- **Consequence:** Wrong buffer mutated — active buffer corrupted during
  inference, or inactive buffer left stale.
- **Suggested fix:** Make `use_secondary_` an `std::atomic<bool>`, or ensure all
  readers hold at least a shared lock.

### 10. `constant_folded_` double-checked locking without atomics

- **Shared state:** `constant_folded_`, `constant_folded_secondary_`
  (`ConstantState` enum, `uint8_t`)
- **Writer:** `run()` writes `FOLDED` after upgrading to unique lock;
  `update_constant_buffer()` writes `INITIALIZED` without any lock
- **Reader:** `run()` reads under shared lock (first check), then unique lock
  (second check)
- **Race scenario:** On weakly-ordered architectures (ARM/aarch64), a
  compiler/CPU could reorder the non-atomic write relative to mutex operations.
  Thread A sees `FOLDED` but reads stale constant data that hasn't been fully
  written yet.
- **Consequence:** Incorrect inference results from reading partially-written
  constants.
- **Suggested fix:** Use `std::atomic<ConstantState>` with acquire/release
  semantics.

### 11. `run()` dangling reference during lock upgrade

- **Shared state:** `constant_folded_` / `constant_folded_secondary_`
- **Issue:** `run()` creates `ConstantState& const_folded` based on
  `use_secondary_` under shared lock. Then unlocks shared, acquires unique.
  Between unlock and re-lock, `swap_constant_buffer()` can flip
  `use_secondary_`, making the reference point to the wrong field.
- **Race scenario:** Thread A selects `const_folded = constant_folded_`
  (primary). Unlocks shared. Thread B swaps to secondary. Thread A acquires
  unique, runs const folding, sets `constant_folded_ = FOLDED` — but the active
  buffer is now secondary, which remains `INITIALIZED`.
- **Consequence:** Constant folding applied to wrong buffer, potential
  double-folding or use of unfolded constants.
- **Suggested fix:** Re-evaluate `use_secondary_` after acquiring unique lock.

### 12. TOCTOU on lazy kernel loading in generated code

- **Shared state:** `static CUfunction kernel_name = nullptr;` emitted by
  codegen in generated CUDA kernel wrappers
- **Writer:** The check-then-load pattern: `if (kernel_name == nullptr) { load; }`
- **Reader:** Subsequent calls read the pointer
- **Race scenario:** Two threads call the same compiled function for the first
  time. Both see `nullptr`, both load the kernel, one overwrites the other's
  stored function pointer. On weakly-ordered architectures, a thread could see
  a non-null but partially-initialized function pointer.
- **Consequence:** Double load (wasteful), or on ARM, a call through a
  partially-visible pointer (crash).
- **Tier:** 2 (multi-thread inference of same compiled function).
- **Suggested fix:** Use `std::call_once` / `std::once_flag`, or
  `std::atomic` with acquire/release.

### 13. TOCTOU on `loadLazyCompileFuncs()` statics

- **Shared state:** `triton_lazy_compile_module` and four other static
  `PyObject*` pointers in `lazy_triton_compile.h`
- **Writer:** `loadLazyCompileFuncs()` checks null, then stores to five pointers
  without synchronization
- **Race scenario:** Currently called from static initializer (serialized by
  `dlopen`), but the pattern is fragile. If ever called from a runtime path,
  two concurrent calls would race on the stores.
- **Consequence:** Stale or partially-initialized function pointers, crash if
  called through them.
- **Suggested fix:** `std::call_once`.

### 14. TOCTOU between `cache_lookup` and `cache_miss`

- **Shared state:** `aoti_kernel_cache_` vector
- **Issue:** `cache_lookup()` returns false (no match). `cache_miss()` assumes
  this still holds. No lock spans the check-then-insert sequence.
- **Race scenario:** Thread A misses cache. Thread B concurrently inserts a
  matching entry. Thread A compiles and inserts a duplicate.
- **Consequence:** Wasted compilation, duplicate entries. Combined with issues
  1/2, the concurrent insertions are UB.
- **Suggested fix:** Hold lock across lookup-then-insert, or re-check under
  write lock.

## Minor Issues

### 15. `get_constant_blob_ptr` lazy allocation is not thread-safe

- **Shared state:** `constant_blob_`, `constant_blob_secondary_` (unique_ptr)
- **Issue:** Check-then-allocate with no locking. Two concurrent callers could
  both allocate, one overwriting the other (memory leak).
- **Suggested fix:** Allocate under lock or use `std::call_once`.

### 16. `get_constants_map`/`get_constants_array` lazy allocation of secondary

- **Shared state:** `constants_map_secondary_`, `constants_array_secondary_`
  (shared_ptr)
- **Issue:** Same check-then-allocate pattern as above.
- **Suggested fix:** Same as above.

### 17. `KernelContextGuard` move constructor leaves dangling TLS pointer

- **Shared state:** Thread-local `kernel_context` pointer
  (`kernel_context_tls.h`)
- **Issue:** Default move constructor moves `owned_context_` to a new address
  without updating TLS pointer. Old destructor clears TLS prematurely.
- **Suggested fix:** Delete move constructor or update TLS in it.

### 18. `run_const_fold` exception path leaves model with wrong constants

- **Shared state:** Model instance's `constants_map_` and `constants_`
- **Issue:** If an exception occurs after temporarily swapping a model's
  constants to the inactive buffer, the catch block returns the model to the
  pool without restoring its original constants.
- **Suggested fix:** RAII guard to restore constants on scope exit.

### 19. `OSSProxyExecutor::call_function` uses non-const `operator[]`

- **Shared state:** `custom_objs_` map in `OSSProxyExecutor`
  (`oss_proxy_executor.cpp:828`)
- **Issue:** `operator[]` is non-const. Concurrent calls are technically UB even
  when no insertion occurs. All concurrent callers would hit the same code path.
- **Suggested fix:** Replace with `.at()` (const-qualified).

### 20. Windows `create_temp_dir` returns shared temp directory

- **Shared state:** System temp directory path
- **Issue:** On Windows, `create_temp_dir()` returns the system temp directory
  without creating a unique subdirectory. Concurrent model loading extracts
  files into the same directory.
- **Suggested fix:** Create unique subdirectory per load operation.

## Summary Table

| # | Issue | Severity | Tier | Location |
|---|-------|----------|------|----------|
| 1 | `aoti_kernel_cache_` concurrent read/write (UAF) | SEVERE | 1/2 | `kernel_holder.cpp:213,472` |
| 2 | Concurrent `push_back` on `aoti_kernel_cache_` | SEVERE | 2 | `kernel_holder.cpp:472` |
| 3 | `update_constant_buffer` races with `run()` | SEVERE | 2 | `model_container.h` |
| 4 | `extract_constants_map` races with updates | SEVERE | 2 | `model_container.h` |
| 5 | `free_inactive_constant_buffer` races with swap | SEVERE | 2 | `model_container.h` |
| 6 | `run_const_fold` inactive buffer race | SEVERE | 2 | `model_container.h` |
| 7 | `load_json_file` static JSON object | SEVERE | 2 | `model_package_loader.cpp:159` |
| 8 | `getAOTIModelRunnerRegistry` unsynchronized map | SEVERE | 1 | `model_container_runner.cpp:388` |
| 9 | `use_secondary_` read without synchronization | Significant | 2 | `model_container.h` |
| 10 | `constant_folded_` double-checked locking w/o atomics | Significant | 2 | `model_container.h` |
| 11 | `run()` dangling reference during lock upgrade | Significant | 2 | `model_container.h` |
| 12 | Lazy kernel loading TOCTOU in generated code | Significant | 2 | codegen pattern |
| 13 | `loadLazyCompileFuncs()` static TOCTOU | Significant | 2 | `lazy_triton_compile.h` |
| 14 | TOCTOU between `cache_lookup` and `cache_miss` | Significant | 2 | `kernel_holder.cpp` |
| 15 | `get_constant_blob_ptr` lazy alloc race | Minor | 2 | `model_container.h` |
| 16 | `get_constants_map` lazy alloc race | Minor | 2 | `model_container.h` |
| 17 | `KernelContextGuard` move ctor TLS | Minor | 2 | `kernel_context_tls.h` |
| 18 | `run_const_fold` exception path | Minor | 2 | `model_container.h` |
| 19 | `OSSProxyExecutor` non-const `operator[]` | Minor | 2 | `oss_proxy_executor.cpp:828` |
| 20 | Windows `create_temp_dir` collision | Minor | 2 | `model_package_loader.cpp` |

## Key Architectural Observations

**AOTI Eager (`kernel_holder.cpp`) has zero synchronization.** The
`AOTIPythonKernelHolder` class is registered in the dispatcher and shared across
all threads. Its `aoti_kernel_cache_` vector is read and written with no locks,
no atomics, nothing. This is the highest-priority fix — issues 1 and 2 are
straightforward crashes under concurrent use.

**AOTI Runtime `model_container.h` has partial locking.** `run()` and
`swap_constant_buffer()` use `model_exec_mutex_`, but the management APIs
(`update_constant_buffer`, `extract_constants_map`, `free_inactive_constant_buffer`)
bypass it entirely. The locking scheme needs to be extended to cover all public
entry points.

**AOTI Torch shim layer is clean.** Pure stateless wrappers around ATen ops.
No shared mutable state. Only one minor issue (non-const `operator[]`).

**`cpp_prefix.h` is clean.** Uses `thread_local` for caches, stateless helper
functions. The generated code patterns (lazy kernel loading) are the concern,
not the prefix itself.
