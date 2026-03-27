# Free-Threading Safety Report: jit/frontend (Part 2)

**Overall Risk: LOW**

## Files Reviewed

- `jit/frontend/function_schema_parser.h`
- `jit/frontend/inline_loop_condition.cpp`
- `jit/frontend/inline_loop_condition.h`
- `jit/frontend/ir_emitter.cpp`
- `jit/frontend/ir_emitter.h`
- `jit/frontend/lexer.cpp`
- `jit/frontend/lexer.h`
- `jit/frontend/mini_environment.h`
- `jit/frontend/name_mangler.cpp`
- `jit/frontend/name_mangler.h`
- `jit/frontend/parse_string_literal.h`
- `jit/frontend/parser.cpp`
- `jit/frontend/parser.h`
- `jit/frontend/parser_constants.h`
- `jit/frontend/resolver.h`

## Summary

These files implement the TorchScript frontend: lexer, parser, IR emitter, and supporting utilities. They operate almost entirely in C++ on JIT IR data structures with no direct Python C API usage (no `PyObject*`, no `PyGILState`, no `Py_BEGIN_CRITICAL_SECTION`). The primary thread-safety considerations are static/global C++ state, which are limited and mostly safe.

## Detailed Findings

### Issue 1: Static local `globals` map containing `shared_ptr` values (ir_emitter.cpp:466)

**Severity: Low** | Category: Static mutable C++ state in Python-facing functions

In `Environment::getSugaredVar()`, a `static std::unordered_map<std::string, SugaredValuePtr> globals` is initialized with a brace-enclosed initializer list. C++11 guarantees thread-safe initialization of function-local statics, so construction is safe.

Post-initialization, this map is only read (via `find`), never mutated. However, the values are `shared_ptr<SugaredValue>` instances. These shared pointers are returned to callers (copied), which increments their reference counts. `std::shared_ptr` reference counting is atomic and thread-safe for concurrent reads/copies. The `SugaredValue` objects themselves appear to be logically immutable after construction.

No fix needed.

### Issue 2: Static local `str_to_kind` map (lexer.cpp:65)

**Severity: Low** | Category: Lazy init pattern

`stringToKind()` contains a `static std::unordered_map<std::string, int> str_to_kind` initialized via a lambda. C++11 guarantees thread-safe initialization. After construction, only `at()` (const read) is called. Safe.

### Issue 3: Static const maps `binary_prec` and `unary_prec` (lexer.cpp:9, 40)

**Severity: Safe** | Category: Static immutable data

These are `static const` maps at file scope, initialized at startup. They are only read by `SharedParserData::isUnary` and `SharedParserData::isBinary`. No thread-safety concern.

### Issue 4: `sharedParserData()` singleton (lexer.cpp:99-102)

**Severity: Low** | Category: Lazy init pattern

Returns a reference to a `static SharedParserData data` local. C++11 guarantees thread-safe initialization. `SharedParserData` is constructed once and its `match()` method is read-only (it traverses the `TokenTrie` without mutation). The `isUnary`/`isBinary`/`isRightAssociative` methods are also read-only. Safe after construction.

### Issue 5: `NameMangler::mangleIndex_` (name_mangler.cpp/h)

**Severity: Safe (instance state)** | Category: Not applicable

`NameMangler` is a class with instance state (`mangleIndex_`). The `mangle()` method mutates `mangleIndex_`. However, `NameMangler` is not a singleton or static; it is instantiated per-use. The `static const std::string manglePrefix` at line 6 of name_mangler.cpp is a function-local static const string, safely initialized.

No thread-safety concern unless a single `NameMangler` instance is shared across threads, which would be a caller responsibility.

### Issue 6: `reportSourceLocation` reads environment variable (ir_emitter.cpp:42-54)

**Severity: Safe** | Category: No mutable state

This anonymous-namespace function reads `PYTORCH_JIT_ENABLE_LARGE_SOURCE_LOCATION` via `c10::utils::get_env`. It has no static mutable state and returns a computed value. Safe.

## Non-Issues

- **`MiniEnvironment` template class** (`mini_environment.h`): Instance-local state only. No global or static mutable data.
- **`Parser` / `ParserImpl`** (`parser.cpp`, `parser.h`): Per-instance state. Each `Parser` owns a `ParserImpl` via `unique_ptr`, which owns a `Lexer`. No shared mutable state. The reference to `sharedParserData()` is read-only.
- **`inline_loop_condition.cpp`**: Pure graph transformation functions operating on passed-in `Graph` objects. No global state.
- **`parse_string_literal.h`**: Pure functions with no state.
- **`resolver.h`**: Abstract interface and simple `NativeResolver` implementation. `nativeResolver()` creates a new instance each time. No global state.
- **`ir_emitter.h`**: Header declaring `runCleanupPasses` and `meaningfulName`. No state.
- **`function_schema_parser.h`**: Header declaring parse functions. No state.
- **`parser_constants.h`**: `static constexpr const char*` -- compile-time constant, safe.
- **`to_ir` struct** (ir_emitter.cpp): Contains per-compilation mutable state (`integral_constants`, `fp_constants`, `complex_constants`, etc.), but this is instance-local, not shared. Each TorchScript function compilation creates its own `to_ir`.

## Conclusion

This batch of files is primarily a self-contained C++ compiler frontend for TorchScript. There is no Python C API usage whatsoever -- no `PyObject*`, no GIL acquisition, no borrowed references from Python containers. All static/global data is either const, safely initialized via C++11 function-local statics, or read-only after initialization. The mutable state is entirely instance-local (per-parser, per-compilation). No changes are required for free-threading safety.
