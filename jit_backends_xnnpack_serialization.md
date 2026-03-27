# Free-Threading Safety Audit: jit/backends/xnnpack/serialization

**Overall Risk: Low**

## Files Reviewed
- `torch/csrc/jit/backends/xnnpack/serialization/serializer.cpp`
- `torch/csrc/jit/backends/xnnpack/serialization/serializer.h`

## Findings

### 1. XNNSerializer -- Per-instance state, no static mutable globals (Safe)
**File:** `serializer.h`, `serializer.cpp`
The `XNNSerializer` class holds per-instance state:
- `_builder` (FlatBufferBuilder)
- `_nodes` (vector of serialized nodes)
- `_values` (vector of serialized values)
- `_constantBuffer` (vector of constant buffers)
- `_bufferSizes` (vector of sizes)
- `_version_sha1` (const string)

All members are per-instance and none are static or shared. The class is used within `XNNGraph` (one serializer per graph builder instance). There is no synchronization, but since each `XNNSerializer` is owned by a single `XNNGraph` and used sequentially during graph building, no races occur.

**Severity:** None

### 2. XNNSerializer methods -- No Python interop, no global side effects (Safe)
**Files:** `serializer.cpp`, all methods
All methods (`serializeAddNode`, `serializeData`, `serializeTensorValue`, `finishAndSerialize`) operate exclusively on instance members. No Python C API calls, no global state access, no static mutable variables.

**Severity:** None

## Summary

This group is entirely safe. `XNNSerializer` is a purely per-instance serialization utility with no static mutable state, no Python interop, and no global side effects. It is used during graph building as a local object within `XNNGraph` and is not shared across threads.
