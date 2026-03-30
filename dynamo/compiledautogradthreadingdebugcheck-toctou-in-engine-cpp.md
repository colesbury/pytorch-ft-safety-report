# `CompiledAutogradThreadingDebugCheck` TOCTOU in engine.cpp

- **Status:** Open
- **Severity:** Minor
- **Tier:** Tier 2
- **Component:** compiled_autograd
- **Source report:** [dynamo_compiled_autograd_v2.md](../dynamo_compiled_autograd_v2.md)

- **Tier:** Tier 2
- **Shared state:** `num_threads_in_compiled_autograd` (atomic int32 in
  engine.cpp).
- **Race scenario:** The check
  `num_threads_in_compiled_autograd.load() == 0` is a TOCTOU race: two
  threads could both see 0, both proceed past the check, and both try to
  acquire the compiled autograd mutex. The mutex (`try_lock`) catches the
  actual conflict, so this is merely a misleading error message, not a
  correctness issue. Additionally, `_thread_check.release()` is called before
  entering `compiled_autograd`, so the counter is decremented before the
  mutex is acquired -- a third thread could slip past the check.
- **Suggested fix:** This is informational only; the mutex provides the real
  protection. Consider removing the debug check or making it accurate.
