# Free-Threading Safety Audit: jit (top-level)

**Overall Risk: Medium**

## Files Reviewed
- `torch/csrc/jit/jit_log.cpp`
- `torch/csrc/jit/jit_log.h`
- `torch/csrc/jit/jit_opt_limit.cpp`
- `torch/csrc/jit/jit_opt_limit.h`
- `torch/csrc/jit/resource_guard.h`

## Findings

### 1. JitLoggingConfig singleton -- Unsynchronized mutable state (Medium)
**File:** `jit_log.cpp`, lines 20-64
```cpp
class JitLoggingConfig {
 public:
  static JitLoggingConfig& getInstance() {
    static JitLoggingConfig instance;
    return instance;
  }
 private:
  std::string logging_levels;
  std::unordered_map<std::string, size_t> files_to_levels;
  std::ostream* out;
  ...
 public:
  void setLoggingLevels(std::string levels) {
    this->logging_levels = std::move(levels);
    parse();
  }
  const std::unordered_map<std::string, size_t>& getFilesToLevels() const {
    return this->files_to_levels;
  }
  void setOutputStream(std::ostream& out_stream) {
    this->out = &out_stream;
  }
  std::ostream& getOutputStream() {
    return *(this->out);
  }
};
```
The `JitLoggingConfig` singleton is a Meyer's singleton (safe construction in C++11+), but its members (`logging_levels`, `files_to_levels`, `out`) are read and written with no synchronization. Specifically:

- `set_jit_logging_levels()` calls `setLoggingLevels()` which mutates `logging_levels` and `files_to_levels`.
- `is_enabled()` calls `getFilesToLevels()` which returns a const reference to the map.
- `JIT_LOG` macro calls `get_jit_logging_output_stream()` which reads `out`.
- `set_jit_logging_output_stream()` writes `out`.

If `set_jit_logging_levels()` is called from one thread while `is_enabled()` or `JIT_LOG` runs on another, this is a data race on the map and the `out` pointer. The `is_enabled()` function is called from every `JIT_LOG` macro invocation, which can happen on any thread during JIT compilation.

**Severity: Medium** -- The setters are typically called at startup or from test code, but there is no enforcement of this. Under free-threading, a concurrent `setLoggingLevels` + `is_enabled` is a genuine data race. The `parse()` method clears and repopulates `files_to_levels` in place, so even a "brief" concurrent call could see a partially-cleared map.

**Recommendation:** Add a `std::shared_mutex` (read-write lock) to the singleton: readers (`getFilesToLevels`, `getOutputStream`) take a shared lock; writers (`setLoggingLevels`, `setOutputStream`) take an exclusive lock. Alternatively, make the configuration immutable after initialization and use atomic swap of a shared_ptr to the config.

### 2. passes_to_current_counter() -- Unprotected mutable static map (Medium)
**File:** `jit_opt_limit.cpp`, lines 14-17
```cpp
static std::unordered_map<std::string, int64_t>& passes_to_current_counter() {
  static std::unordered_map<std::string, int64_t> passes_to_current_counter;
  return passes_to_current_counter;
}
```
This function-local static map is returned by mutable reference and is modified in `opt_limit()` (lines 66-76):
```cpp
passes_to_current_counter().insert({pass, 0});
...
current_count_it->second++;
```
These reads and writes happen with no synchronization. If multiple threads invoke JIT optimization passes concurrently, they will race on this map -- both the `find`/`insert` operations and the counter increment.

**Severity: Medium** -- This is a mutable global counter map accessed from JIT optimization passes. Under free-threading with concurrent JIT compilations, this is a data race.

**Recommendation:** Protect with a `std::mutex`, or use a `std::atomic<int64_t>` for the counters (though the map itself still needs protection for insertions).

### 3. Static local maps in opt_limit() -- Safe (read-only after init)
**File:** `jit_opt_limit.cpp`, lines 48-55
```cpp
static const auto opt_limit = c10::utils::get_env("PYTORCH_JIT_OPT_LIMIT");
...
static const std::unordered_map<std::string, int64_t> passes_to_opt_limits =
    parseJITOptLimitOption(opt_limit.value());
```
These are `static const` / `static const auto` locals, initialized once (thread-safely in C++11+) and never mutated. Safe.

**Severity:** None

### 4. ResourceGuard -- Safe
**File:** `resource_guard.h`
`ResourceGuard` is a simple RAII guard with a `std::function` destructor and a `bool _released` flag. It is a stack-local utility object, not shared between threads. No thread-safety concern.

**Severity:** None

### 5. No Python C API usage
None of these files use the Python C API. They are pure C++ JIT infrastructure.

## Summary

The two main issues are:

1. **JitLoggingConfig singleton** has unsynchronized mutable state (logging levels map, output stream pointer) that can be read and written concurrently. The `is_enabled()` function is called from JIT_LOG macros on potentially any thread.

2. **passes_to_current_counter** is a mutable global map modified without synchronization from `opt_limit()`, which is called from JIT optimization passes.

Both are classic cases of global mutable state that was "safe enough" under the assumption of GIL-serialized or single-threaded JIT compilation, but becomes a real data race under free-threading or concurrent JIT compilation scenarios.
