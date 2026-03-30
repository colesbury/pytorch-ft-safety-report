# `GuardManager::_dict_tag` updated without synchronization across roots watching the same dict

- **Status:** Open
- **Severity:** Significant
- **Tier:** Tier 2
- **Component:** guards
- **Source report:** [dynamo_guards_v2.md](../dynamo_guards_v2.md)

- **Tier:** Tier 2
- **Shared state:** `GuardManager::_dict_tag` (uint64_t, line 3709)
- **Writer(s):** `check_accessors_nopybind` (line 3547): `_dict_tag = new_tag;` -- written at the end of successful accessor checks.
- **Reader(s):** `check_accessors_nopybind` (line 3507): `matches_dict_tag = (new_tag == _dict_tag);` -- read at the start.
- **Race scenario:** This is protected by `RootGuardManager::_lock` for a given root, so concurrent evaluations of the same root are safe. However, if the same `GuardManager` instance were reachable from two different roots (not the current architecture for non-leaf managers, but worth noting), the read and write would race. Under the current architecture, this is safe because non-root `GuardManager` instances are owned (via `unique_ptr`) by a single root.
- **Consequence:** Benign stale read (wrong tag match decision, causing unnecessary full traversal).
- **Note:** This is safe under the current ownership model but fragile if the architecture changes.
