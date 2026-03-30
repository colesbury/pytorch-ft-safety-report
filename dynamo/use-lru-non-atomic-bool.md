# `use_lru` non-atomic bool

- **Status:** Open
- **Severity:** Minor
- **Tier:** Tier 2
- **Component:** eval_frame
- **Source report:** [dynamo_eval_frame_v2.md](../dynamo_eval_frame_v2.md)

- **Shared state:** `use_lru` -- a file-scope `bool` in `extra_state.cpp`
  (line 17).
- **Writer(s):** `_set_lru_cache()` -- from Python.
- **Reader(s):** `lookup()` (line 197) and `create_cache_entry()` (line 212)
  -- on every frame evaluation / cache creation.
- **Race scenario:** Stale read. Same as issue #11. A stale read might cause
  one lookup to skip or perform an unnecessary `move_to_front`, which is
  benign.
- **Tier:** **Tier 2**.
- **Suggested fix:** Make it `std::atomic<bool>`. Low priority.
