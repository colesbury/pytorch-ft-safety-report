# Profiler CodeLocation stores borrowed const char* from code objects

- **Status:** Open
- **Severity:** Minor
- **Tier:** Tier 2
- **Component:** profiler_python.cpp

## Shared state

The `CodeLocation` struct stores raw `const char*` pointers obtained from
Python code objects:

```cpp
struct CodeLocation {
    const char* filename_{nullptr};
    const char* name_{nullptr};
    int line_number_{0};
};
```

These pointers are obtained via `THPUtils_unpackStringView(code->co_filename).data()`.
They point into the internal storage of Python string objects (`co_filename`
and `co_name` on the code object).

## Writers

The `CodeLocation` constructor (called when a new call site is first
encountered in `recordPyCall`) stores these borrowed pointers.

## Readers

The `ValueCache` stores and loads `CodeLocation` instances. During post-
processing, `trimPrefixes()` reads the `filename_` pointer as a C string.
The `hash` and `operator==` functions dereference these pointers.

## Race scenario

Under free-threading, code objects are shared across threads. If a code
object is garbage collected (because the function that owned it was deleted
or reassigned) while the profiler still holds a `CodeLocation` with raw
pointers into its strings, the pointers dangle. Subsequent access (during
post-processing or hashing) would read freed memory.

In practice, code objects referenced by active frames are kept alive, but
the profiler may hold `CodeLocation` entries for code objects from frames
that have since returned and been garbage collected.

## Suggested fix

Either hold a strong reference to the `PyCodeObject*` (via `Py_INCREF`)
alongside each `CodeLocation`, or copy the filename and name strings into
owned `std::string` members. The latter is simpler and avoids Python
refcounting overhead during profiling.
