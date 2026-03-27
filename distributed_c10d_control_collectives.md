# Free-Threading Safety Audit: distributed_c10d_control_collectives

## Overall Risk: LOW

## Files Analyzed
- `distributed/c10d/control_collectives/StoreCollectives.cpp`

## Detailed Analysis

### 1. StoreCollectives class - per-instance state only
- **Location**: Throughout the file
- **Issue**: `StoreCollectives` is a concrete implementation of `ControlCollectives` that delegates all operations to an underlying `Store`. It holds:
  - `store_`: an intrusive_ptr to a Store (shared, thread-safe via Store's own synchronization)
  - `rank_`, `worldSize_`: immutable after construction
  - `seenKeys_`: a `std::unordered_set<std::string>` used in `enforceUnique()`
- **Risk**: LOW

### 2. `enforceUnique()` and `seenKeys_`
- **Location**: Lines 215-220
- **Issue**: `seenKeys_` is an unprotected `std::unordered_set`. If multiple threads call collective operations (e.g., `barrier`, `allGather`) concurrently on the same `StoreCollectives` instance, they would race on `seenKeys_`. However, `StoreCollectives` is designed for sequential collective usage (each key is used exactly once), so concurrent calls on the same instance are a user error.
- **Risk**: LOW - Would only manifest if the same `StoreCollectives` instance is used concurrently, which violates its design contract.

### 3. No Python C API usage
- The file is pure C++ with no Python object manipulation.
- **Risk**: NONE

### 4. No global mutable state
- All state is per-instance. No static/global variables.
- **Risk**: NONE

## Summary

This file has no free-threading safety concerns. All state is instance-local, and the underlying `Store` handles its own thread safety. The only theoretical issue is concurrent access to `seenKeys_`, but the class's usage contract prohibits that.
