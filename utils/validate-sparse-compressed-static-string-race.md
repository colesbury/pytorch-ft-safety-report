# `static std::string sig` data race in `_validate_sparse_compressed_tensor_args_template`

- **Status:** Open
- **Severity:** Significant
- **Tier:** Tier 2
- **Component:** tensor_new
- **Source reports:** [utils__part4.md](../utils__part4.md)

- **Shared state:** `static std::string sig` inside the template function
  `_validate_sparse_compressed_tensor_args_template<required_layout>()` in
  `torch/csrc/utils/tensor_new.cpp` (line 1385).
- **Writer(s):**
  - Every call to the function writes to `sig` via `sig = "..."` inside
    a switch statement (lines 1387-1403). Since `required_layout` is a
    template parameter, each instantiation always writes the same value,
    but the write happens on every call, not just the first.
- **Reader(s):**
  - `static PythonArgParser parser({sig})` (line 1405) reads `sig`
    during its one-time initialization on the first call.
  - The `sig` variable is also implicitly read during the assignment
    (self-assignment of `std::string` reads internal state).
- **Race scenario:** Two threads concurrently call
  `_validate_sparse_csr_tensor_args` (which instantiates the template
  with `SparseCsr`). Both execute `sig = "_validate_sparse_csr_tensor(...)"`
  on line 1388. Even though both write the same value, concurrent writes
  to `std::string` are undefined behavior -- `std::string::operator=`
  reads and modifies internal pointer, size, and capacity fields, and
  two concurrent assignments can corrupt the internal state (e.g.,
  double-free of the heap buffer if the string exceeds SSO capacity,
  or a torn read of the size field). This is a genuine data race on
  the `std::string` internals.
- **Suggested fix:** Remove the `static` qualifier from `sig` and make
  it a plain local variable. Since `required_layout` is a compile-time
  constant, each instantiation always picks the same branch. Better yet,
  use `if constexpr` to return a `const char*` string literal:
  ```cpp
  constexpr const char* sig = [] {
      if constexpr (required_layout == c10::Layout::SparseCsr)
          return "_validate_sparse_csr_tensor(...)";
      // ...
  }();
  static PythonArgParser parser({sig});
  ```
  This eliminates the mutable static entirely.
