# `constant_folded_` double-checked locking without atomics

- **Status:** Open
- **Severity:** Significant
- **Component:** aoti_runtime/model_container.h

## Missing atomics

- **Shared state:** `constant_folded_`, `constant_folded_secondary_`
  (`ConstantState` enum, `uint8_t`)
- **Writer:** `run()` writes `FOLDED` after upgrading to unique lock;
  `update_constant_buffer()` writes `INITIALIZED` without any lock.
- **Reader:** `run()` reads under shared lock (first check), then unique lock
  (second check).
- **Race scenario:** On weakly-ordered architectures (ARM/aarch64), a
  compiler/CPU could reorder the non-atomic write relative to mutex operations.
  Thread A sees `FOLDED` but reads stale constant data that hasn't been fully
  written yet.
- **Suggested fix:** Use `std::atomic<ConstantState>` with acquire/release
  semantics.

## Dangling reference during lock upgrade

- **Shared state:** Same `constant_folded_` / `constant_folded_secondary_`
- **Issue:** `run()` creates `ConstantState& const_folded` based on
  `use_secondary_` under shared lock. Then unlocks shared, acquires unique.
  Between unlock and re-lock, `swap_constant_buffer()` can flip
  `use_secondary_`, making the reference point to the wrong field.
- **Race scenario:** Thread A selects `const_folded = constant_folded_`
  (primary). Unlocks shared. Thread B swaps to secondary. Thread A acquires
  unique, runs const folding, sets `constant_folded_ = FOLDED` — but the active
  buffer is now secondary, which remains `INITIALIZED`.
- **Consequence:** Constant folding applied to wrong buffer, potential
  double-folding or use of unfolded constants.
- **Suggested fix:** Re-evaluate `use_secondary_` after acquiring unique lock.
