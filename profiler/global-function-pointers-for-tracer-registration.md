# Global function pointers for tracer registration are non-atomic

- **Status:** Open
- **Severity:** Significant
- **Component:** profiler/orchestration/python_tracer.cpp

- **Shared state:** `make_fn` and `memory_make_fn` -- plain function pointers
  (lines 5-6) of types `MakeFn` and `MakeMemoryFn`.
- **Writer(s):**
  - `registerTracer()` (line 33): writes `make_fn` during Python module init
    (called from `torch::autograd::profiler::python_tracer::init()`).
  - `registerMemoryTracer()` (line 44): writes `memory_make_fn` during the
    same init path.
- **Reader(s):**
  - `PythonTracerBase::make()` (line 37): reads `make_fn` when starting the
    profiler. Checks for null then calls through the pointer.
  - `PythonMemoryTracerBase::make()` (line 48): same pattern for
    `memory_make_fn`.
- **Race scenario:** Under free-threading, Python module imports can run
  concurrently. If `torch.autograd` is being imported on one thread (calling
  `registerTracer`) while another thread starts the profiler (calling
  `PythonTracerBase::make`), the read of `make_fn` races with the write.
  For a plain pointer this is technically UB, though in practice function
  pointers are pointer-sized and the worst case is reading null or the
  correct value (torn reads are unlikely on aligned pointer writes).
  Still, it should be fixed for correctness.
- **Suggested fix:** Use `std::atomic<MakeFn>` and `std::atomic<MakeMemoryFn>`.
  The load/store overhead is negligible since these are only accessed during
  profiler start.
