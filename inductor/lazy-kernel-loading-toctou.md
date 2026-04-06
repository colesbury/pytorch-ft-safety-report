# TOCTOU on lazy kernel loading in generated code

- **Status:** Open
- **Severity:** Significant
- **Component:** codegen pattern (static_launcher/)

- **Shared state:** `static CUfunction kernel_name = nullptr;` emitted by
  codegen in generated CUDA kernel wrappers
- **Writer:** The check-then-load pattern: `if (kernel_name == nullptr) { load; }`
- **Reader:** Subsequent calls read the pointer.
- **Race scenario:** Two threads call the same compiled function for the first
  time. Both see `nullptr`, both load the kernel, one overwrites the other's
  stored function pointer. On weakly-ordered architectures, a thread could see
  a non-null but partially-initialized function pointer.
- **Consequence:** Double load (wasteful), or on ARM, a call through a
  partially-visible pointer (crash).
- **Tier:** 2 (multi-thread inference of same compiled function).
- **Suggested fix:** Use `std::call_once` / `std::once_flag`, or
  `std::atomic` with acquire/release.
