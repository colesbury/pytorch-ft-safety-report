# Free-Threading Safety Audit: profiler_unwind__part1

**Overall Risk: LOW**

## Files Reviewed
- `profiler/unwind/action.h`
- `profiler/unwind/communicate.h`
- `profiler/unwind/debug_info.h`
- `profiler/unwind/dwarf_enums.h`
- `profiler/unwind/dwarf_symbolize_enums.h`
- `profiler/unwind/eh_frame_hdr.h`
- `profiler/unwind/fast_symbolizer.h`
- `profiler/unwind/fde.h`
- `profiler/unwind/lexer.h`
- `profiler/unwind/line_number_program.h`
- `profiler/unwind/mem_file.h`
- `profiler/unwind/range_table.h`
- `profiler/unwind/sections.h`
- `profiler/unwind/unwind.cpp`
- `profiler/unwind/unwind.h`
- `profiler/unwind/unwind_error.h`
- `profiler/unwind/unwind_fb.cpp`
- `profiler/unwind/unwinder.h`

## Detailed Findings

### Issue 1 -- `shared_timed_mutex` for library cache (unwind.cpp)

**Severity: LOW**
**Location:** `unwind.cpp`, lines ~57-87

The unwinder uses a `std::shared_timed_mutex` with an `UpgradeExclusive` pattern:

```cpp
struct UpgradeExclusive {
  UpgradeExclusive(std::shared_lock<std::shared_timed_mutex>& rdlock)
      : rdlock_(rdlock) {
    rdlock_.unlock();
    rdlock_.mutex()->lock();
  }
  ~UpgradeExclusive() {
    rdlock_.mutex()->unlock();
    rdlock_.lock();
  }
};
```

This is a manual upgrade-to-exclusive pattern. The unlock-then-lock sequence is not atomic, meaning another thread could acquire the exclusive lock in between. However, the code that uses this pattern appears to handle re-validation after upgrade (checking if the cache was populated by another thread). This is a standard double-checked locking pattern and is safe.

### Issue 2 -- Static symbolizer and frame cache in `unwind_fb.cpp`

**Severity: LOW**
**Location:** `unwind_fb.cpp`, lines ~11-13

```cpp
static std::mutex symbolize_mutex;
static llvm::symbolize::LLVMSymbolizer symbolizer;
static ska::flat_hash_map<void*, Frame> frame_map_;
```

All three statics are protected by `symbolize_mutex` via `std::lock_guard`. This is correctly synchronized. The mutex serializes all symbolization calls, which is appropriate since symbolization is a slow path anyway.

### Issue 3 -- Header-only data structures (most .h files)

**Severity: LOW**

The header files in the unwind directory (`action.h`, `communicate.h`, `debug_info.h`, `dwarf_enums.h`, `eh_frame_hdr.h`, `fast_symbolizer.h`, `fde.h`, `lexer.h`, `line_number_program.h`, `mem_file.h`, `range_table.h`, `sections.h`, `unwinder.h`) define data structures and parsers for DWARF debug information and ELF sections. These are used as local objects during unwind/symbolize operations -- they are not shared across threads. No global mutable state.

### Issue 4 -- `unwind()` function stack walking

**Severity: LOW**
**Location:** `unwind.cpp`

The `unwind()` function walks the calling thread's stack frame. Stack walking is inherently per-thread (each thread has its own stack). The `unwind_entry`/`unwind_c` assembly routines operate on the current thread's registers. Safe under free-threading.

The library metadata cache used during unwinding is protected by the `shared_timed_mutex` noted in Issue 1.

### Issue 5 -- `libraryFor()` and shared library list

**Severity: LOW**
**Location:** `unwind.cpp`

The `libraryFor()` function queries the loaded library list. On Linux, `dl_iterate_phdr` is used to enumerate loaded shared libraries. This is a system call that is thread-safe. The results may be cached in the shared library info structures protected by the reader-writer lock.

## Summary

The unwind subsystem is well-designed for concurrent use. The library metadata cache uses a reader-writer lock (`shared_timed_mutex`), the symbolizer uses a mutex, and stack walking is inherently per-thread. All header-only structures are used as local variables. No free-threading concerns identified.
