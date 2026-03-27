# Free-Threading Safety Audit: jit_operator_upgraders

## Files Reviewed
- `torch/csrc/jit/operator_upgraders/upgraders.cpp`
- `torch/csrc/jit/operator_upgraders/upgraders.h`
- `torch/csrc/jit/operator_upgraders/upgraders_entry.cpp`
- `torch/csrc/jit/operator_upgraders/upgraders_entry.h`
- `torch/csrc/jit/operator_upgraders/utils.cpp`
- `torch/csrc/jit/operator_upgraders/utils.h`
- `torch/csrc/jit/operator_upgraders/version_map.cpp`
- `torch/csrc/jit/operator_upgraders/version_map.h`

## Overview

This group implements operator version upgraders for TorchScript model backward compatibility. It maintains a map from operator names to upgrader graphs and a version map for determining which upgrader to apply. No Python C API is used.

## Issues

### Issue 1: `UpgradersMap` is properly synchronized

**Location:** `upgraders.cpp`, `upgraders.h`

The `UpgradersMap` class uses a `std::mutex lock` to protect `content_` and `isPopulated`. All public methods acquire the lock before accessing shared state. The `set_content` method uses the lock to ensure the map is populated only once.

**Risk:** None.

However, note that `get_content()` returns a `const&` to `content_`, meaning callers hold a reference to data that could theoretically be modified by `test_only_set_content` or `test_only_remove_content` (which also acquire the lock but modify the map). In practice, `set_content` is called once, and test-only methods are not called in production.

### Issue 2: `operatorVersionMap` and `isVersionMapSorted` are unprotected globals

**Location:** `version_map.cpp`, lines 12-106

```cpp
static bool isVersionMapSorted = false;
static std::unordered_map<std::string, std::vector<UpgraderEntry>> operatorVersionMap({...});
```

The `get_operator_version_map()` function:
```cpp
const std::unordered_map<std::string, std::vector<UpgraderEntry>>&
get_operator_version_map() {
  if (!isVersionMapSorted) {
    for (auto entry : operatorVersionMap) {
      std::sort(entry.second.begin(), entry.second.end(), ...);
    }
    isVersionMapSorted = true;
  }
  return operatorVersionMap;
}
```

This has multiple issues:
1. `isVersionMapSorted` is a non-atomic `bool` read/written without synchronization.
2. The sorting loop iterates over `operatorVersionMap` by value (`auto entry` not `auto& entry`), so it actually sorts *copies* and discards them -- the map itself is never sorted! This appears to be a pre-existing bug.
3. Even if the sorting were correct (using references), concurrent calls to `get_operator_version_map()` could race on both the flag and the map mutation.

Additionally, `test_only_add_entry`, `test_only_remove_entry`, and `test_only_reset_flag` modify `operatorVersionMap` and `isVersionMapSorted` without any synchronization.

**Risk:** Medium -- the sorting code is buggy (sorts copies, not the actual map), so `isVersionMapSorted` is effectively just a flag that gates a no-op. But the test-only mutation functions are completely unsynchronized.

**Recommended Fix:** Fix the sorting bug (use `auto&` instead of `auto`), then protect `get_operator_version_map` with `std::call_once` or a mutex. Make `isVersionMapSorted` atomic or eliminate it in favor of `call_once`.

### Issue 3: `calculatePackageVersionBasedOnUpgraders` non-atomic global

**Location:** `version_map.cpp`, lines 122-130

```cpp
static bool calculatePackageVersionBasedOnUpgraders = false;

void calculate_package_version_based_on_upgraders(bool val) {
  calculatePackageVersionBasedOnUpgraders = val;
}

bool get_version_calculator_flag() {
  return calculatePackageVersionBasedOnUpgraders;
}
```

Non-atomic bool read/written from potentially multiple threads.

**Risk:** Low -- this is a simple configuration flag; boolean read/write is typically atomic on common architectures, but it is technically undefined behavior in C++.

**Recommended Fix:** Use `std::atomic<bool>`.

### Issue 4: `populate_upgraders_graph_map` has a TOCTOU race

**Location:** `upgraders_entry.cpp`, lines 141-146

```cpp
void populate_upgraders_graph_map() {
  if (!is_upgraders_map_populated()) {
    auto graphs = generate_upgraders_graph();
    populate_upgraders_map(std::move(graphs));
  }
}
```

The check `is_upgraders_map_populated()` and the subsequent `populate_upgraders_map()` are not atomic as a unit. Two threads could both see `false`, both generate graphs, and both call `populate_upgraders_map`. However, `UpgradersMap::set_content` has an internal guard (`if (isPopulated) return;`) under its mutex, so the second call is a no-op. The wasted work is the only cost.

**Risk:** Low -- correctly handled by the internal guard in `set_content`.

### Issue 5: `kUpgradersEntryMap` is a const static map

**Location:** `upgraders_entry.cpp`, line 16

```cpp
static std::unordered_map<std::string, std::string> kUpgradersEntryMap({...});
```

Initialized at static init time, never modified. Thread-safe.

**Risk:** None.

## Summary

The `UpgradersMap` class itself is properly synchronized with a mutex. The main issues are:

1. **`operatorVersionMap` sorting race** (Issue 2) -- the sorting code appears buggy (sorts copies), and the unsorted flag is racy. Needs a `call_once` or mutex.
2. **`calculatePackageVersionBasedOnUpgraders`** (Issue 3) -- non-atomic bool, trivially fixable.
3. **Test-only functions** modify global state without synchronization, but these are only used in tests.

No Python C API usage in any of these files.
