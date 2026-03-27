# Free-Threading Safety Audit: mtia_profiler

**Overall Risk: None**

## Files Reviewed
- `mtia/profiler/MTIAMemoryProfiler.cpp`
- `mtia/profiler/MTIAMemoryProfiler.h`

## Findings

No free-threading issues found.

## Summary

The `mtia_profiler` module implements `MTIAMemoryProfiler`, a simple class that delegates to `at::detail::getMTIAHooks()` for memory profiling operations (start, stop, export). The `initMemoryProfiler` function registers a factory via `registerMemoryTracer` and is called once during initialization. No mutable static or global state is maintained in these files. Each profiler instance is independent.
