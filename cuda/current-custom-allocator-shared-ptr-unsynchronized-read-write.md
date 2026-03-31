# `current_custom_allocator` shared_ptr unsynchronized read/write

- **Status:** Open
- **Severity:** Minor
- **Tier:** Tier 1
- **Component:** CUDAPluggableAllocator

- **Shared state:** `std::shared_ptr<CUDAAllocator> current_custom_allocator` (CUDAPluggableAllocator.cpp line 365), a namespace-scope global.
- **Writer(s):**
  - `changeCurrentAllocator` (line 383): called from Python when the user installs a custom allocator. Writes both the global `current_custom_allocator` and the `c10::cuda::CUDACachingAllocator::allocator` atomic pointer.
- **Reader(s):**
  - `custom_raw_deleter` (line 393): reads `current_custom_allocator` to call `raw_delete`. This function is the deleter stored in every `DataPtr` allocated by the pluggable allocator. It fires when a tensor's refcount drops to zero, which can happen on any thread.
  - `getCurrentAllocator` (line 368): reads and copies the `shared_ptr`.
- **Race scenario:** Thread A is deallocating a CUDA tensor (e.g., during garbage collection or at scope exit). This invokes `custom_raw_deleter`, which reads `current_custom_allocator` to get the allocator pointer. Concurrently, Thread B calls `changeCurrentAllocator` to swap in a different allocator, writing to the same `shared_ptr`. `std::shared_ptr` is not thread-safe for concurrent read and write of the same instance -- this is undefined behavior. The shared_ptr's internal control block can be corrupted, leading to a crash or use-after-free of the old allocator object.
- **Consequence:** Crash (corrupted shared_ptr internals), or use-after-free if the old allocator is destroyed while `raw_delete` is in progress.
- **Suggested fix:** Use `std::atomic<std::shared_ptr<...>>` (C++20) or protect access with a mutex. Alternatively, since `changeCurrentAllocator` already checks that the allocator is not initialized (i.e., it should only be called once before any allocations), the practical risk is low -- but the code does not enforce this ordering and the `shared_ptr` race is still technically UB. At minimum, the read in `custom_raw_deleter` should use `std::atomic_load` and the write in `changeCurrentAllocator` should use `std::atomic_store` on the shared_ptr.
