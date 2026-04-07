# Static local `PyObject*` from `PyUnicode_InternFromString` in hot path

- **Status:** Open
- **Severity:** Minor
- **Tier:** Tier 2
- **Component:** compiled_autograd
- **Source report:** [dynamo_compiled_autograd_v2.md](../dynamo_compiled_autograd_v2.md)

- **Tier:** Tier 2
- **Shared state:** `static PyObject* method_name` locals in
  `call_begin_capture` (line 803), `call_end_capture` (line 857),
  `ClosingTHPObjectPtr::~ClosingTHPObjectPtr` (line 874), and the `static bool
  flag` lambda (line 1077).
- **Race scenario:** C++11 guarantees thread-safe initialization of
  function-local statics. However, the `static bool flag` lambda at line 1077
  calls `bind_function` which calls into Python. If another thread is blocked
  on the same static initialization guard while Python code tries to re-enter
  this initialization path, a deadlock could occur. The
  `PyUnicode_InternFromString` calls are safe as they are idempotent and
  internally thread-safe in 3.14t.
- **Suggested fix:** Move one-time initialization to module init, or to a
  `std::call_once` with the compiled autograd mutex held.
