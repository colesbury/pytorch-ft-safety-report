# `global_dict_version_id` non-atomic increment

- **Status:** FIXED ([#178703](https://github.com/pytorch/pytorch/pull/178703))
- **Severity:** Minor
- **Tier:** Tier 1
- **Component:** guards
- **Source report:** [dynamo_guards_v2.md](../dynamo_guards_v2.md)

- **Tier:** Tier 1
- **Shared state:** `static uint64_t global_dict_version_id = 1` (line 880)
- **Writer(s):** `dict_version_watch_callback` (line 889): `dict_version_map[dict] = global_dict_version_id++`. `get_dict_version_unchecked` (line 903): `dict_version_map[dict] = global_dict_version_id++`.
- **Reader(s):** Same increment expressions.
- **Race scenario:** Even if the `dict_version_map` is protected by a mutex (per fix for issue #1), the `global_dict_version_id++` is a non-atomic read-modify-write. Two threads could read the same value, both increment to the same next value, and assign the same "version" to two different dicts. This violates the invariant that each dict mutation gets a unique version ID.
- **Consequence:** Two dicts could end up with the same version, causing a guard to incorrectly consider a dict unchanged. This would cause wrong compiled code to execute (incorrect guard pass).
- **Suggested fix:** Make `global_dict_version_id` a `std::atomic<uint64_t>` and use `fetch_add(1)`.
