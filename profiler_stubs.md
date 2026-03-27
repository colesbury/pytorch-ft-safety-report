# Free-Threading Safety Audit: profiler_stubs

**Overall Risk: LOW**

## Files Reviewed
- `profiler/stubs/base.cpp`
- `profiler/stubs/base.h`
- `profiler/stubs/cuda.cpp`
- `profiler/stubs/itt.cpp`

## Detailed Findings

### Issue 1 -- Global profiler stub pointers (base.cpp)

**Severity: LOW**
**Location:** `base.cpp`, lines ~55-81

The `REGISTER_DEFAULT` macro generates, for each stub type (cuda, itt, privateuse1), code like:

```cpp
inline const ProfilerStubs*& cuda_stubs() {
  static const ProfilerStubs* stubs_ =
      static_cast<const ProfilerStubs*>(default_cuda_stubs_addr);
  return stubs_;
}

const ProfilerStubs* cudaStubs() {
  return cuda_stubs();
}

void registerCUDAMethods(ProfilerStubs* stubs) {
  cuda_stubs() = stubs;
}
```

The function-local `static const ProfilerStubs* stubs_` is initialized with a constant (safe), but `registerCUDAMethods` writes to it and `cudaStubs()` reads from it without synchronization. This is technically a data race under free-threading.

However, registration happens during static initialization (e.g., `RegisterCUDAMethods reg;` in `cuda.cpp`) which runs before `main()`. After that, the pointer is only read. The C++ standard guarantees static initialization of translation units completes before `main()`, so by the time any Python thread could call `cudaStubs()`, the pointer is already set and never modified again.

**Recommendation:** No change needed for the current usage pattern. If registration could ever happen after program startup (unlikely), wrapping in `std::atomic<const ProfilerStubs*>` would be a simple fix.

### Issue 2 -- `RegisterCUDAMethods` static initializer (cuda.cpp)

**Severity: LOW**
**Location:** `cuda.cpp`, lines ~121-128

```cpp
struct RegisterCUDAMethods {
  RegisterCUDAMethods() {
    static CUDAMethods methods;
    registerCUDAMethods(&methods);
  }
};
RegisterCUDAMethods reg;
```

Static initialization, runs once before `main()`. The `static CUDAMethods methods` is safely initialized. Safe.

### Issue 3 -- `RegisterITTMethods` static initializer (itt.cpp)

**Severity: LOW**
**Location:** `itt.cpp`, lines ~42-48

Same pattern as Issue 2. Safe.

### Issue 4 -- `CUDAMethods` is stateless (cuda.cpp)

**Severity: LOW**

The `CUDAMethods` struct has no mutable state -- all methods are `const`. It delegates to CUDA runtime APIs which have their own thread-safety guarantees. The `ProfilerStubs` interface itself is read-only after registration. Safe.

## Summary

The profiler stubs are effectively write-once-read-many. Registration happens during static initialization before any Python threads exist. After that, all access is read-only through `const ProfilerStubs*` pointers. No free-threading concerns.
