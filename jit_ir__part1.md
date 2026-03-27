# Free-Threading Safety Report: `torch/csrc/jit/ir/` (Part 1)

**Overall Risk: LOW**

These files implement the JIT intermediate representation (IR) -- the Graph, Node, Value, Block data structures, alias analysis, constants, the IR parser, and supporting utilities. The code is overwhelmingly internal C++ data structures that are not directly exposed to Python threads and do not use the Python C API. The comment in `ir.h` line 153 explicitly states: "like much of graph, it isn't safe for different threads to access the same graph." The JIT IR has never been designed for concurrent access, and Graph objects are not shared across Python threads in normal usage.

There are a small number of static mutable objects worth noting but they present minimal risk.

---

## Files Reviewed

- `jit/ir/alias_analysis.cpp` / `.h`
- `jit/ir/attributes.cpp` / `.h`
- `jit/ir/constants.cpp` / `.h`
- `jit/ir/graph_node_list.h`
- `jit/ir/graph_utils.cpp` / `.h`
- `jit/ir/ir.cpp` / `.h`
- `jit/ir/ir_views.h`
- `jit/ir/irparser.cpp` / `.h`
- `jit/ir/named_value.h`

---

## Detailed Findings

### 1. Static `SourceRange` in `fakeRange()` (ir.cpp:1666-1669) -- Low Risk

```cpp
static inline const SourceRange& fakeRange() {
  static SourceRange range(std::make_shared<Source>(std::string("")), 0, 1);
  return range;
}
```

This is a function-local static that is initialized once and returned by const reference. C++11 guarantees thread-safe initialization of function-local statics. The returned object is effectively immutable after construction. **No fix needed.**

### 2. Static `std::locale` in `Value::setDebugName()` (ir.cpp:885) -- Low Risk

```cpp
static std::locale c_locale("C");
ss.imbue(c_locale);
```

Function-local static, thread-safe initialization guaranteed by C++11. `std::locale` objects are immutable after construction and `imbue` only reads the locale. **No fix needed.**

### 3. Static `constexpr` array in `toString(AttributeKind)` (attributes.h:36) -- No Risk

```cpp
static constexpr const char* names[] = {...};
```

Compile-time constant array. Immutable. **No issue.**

### 4. `Wrap<T>::clear_cb` function pointer (ir.h:155-166) -- Low Risk

The `Wrap<T>` template stores a `clear_cb` function pointer and an `elem` pointer used for invalidation of Python wrappers when C++ objects are destroyed. The comment at line 153 acknowledges this is not thread-safe: "like much of graph, it isn't safe for different threads to access the same graph." This is an existing design constraint of the JIT IR, not a free-threading regression.

### 5. Mutable `op_` cache on `Node` (ir.h:331, ir.cpp:1086-1097) -- Low Risk

```cpp
mutable const Operator* op_{nullptr};
```

`Node::maybeOperator()` lazily populates this cache. Since Graph/Node objects are not shared across Python threads in practice, this is not a free-threading concern. If concurrent access to a Graph were ever introduced, this would need synchronization, but that is a broader architectural change.

### 6. `mapped_mutable_types_` mutable cache on `AliasDb` (alias_analysis.h:300) -- Low Risk

```cpp
mutable ska::flat_hash_map<TypePtr, AliasTypeSet> mapped_mutable_types_;
```

This mutable cache is populated during alias analysis from const methods. `AliasDb` instances are created locally and not shared across threads, so this presents no free-threading concern.

### 7. No Python C API Usage -- No Risk

None of the reviewed files call Python C API functions (no `PyObject*`, `PyGILState_Ensure`, `Py_BEGIN_CRITICAL_SECTION`, etc.). The only Python-related type reference is the `THPObjectPtr` type alias in `ir.h` line 33, which is a forward declaration used by `PythonOp` -- the actual implementation lives in `python_ir.cpp` (not in this audit scope).

### 8. No Global Mutable `PyObject*` -- No Risk

No global or static mutable `PyObject*` variables exist in any of these files.

### 9. IR Parser (`irparser.cpp`) -- No Risk

The `IRParser` class is instantiated locally within `parseIR()` function calls. No shared mutable state.

### 10. Alias Analysis (`alias_analysis.cpp`) -- No Risk

The `AliasDb` class is constructed, used, and destroyed within a single thread's scope. All mutable state is instance-level. No static mutable state.

---

## Summary

These files are pure C++ JIT infrastructure with no direct Python C API interaction. The data structures (Graph, Node, Value, Block, AliasDb) are inherently single-threaded by design and are not shared across Python threads. The few static variables (`fakeRange`, `c_locale`) are either immutable after initialization or protected by C++11 static initialization guarantees. No changes are needed for free-threading safety.
