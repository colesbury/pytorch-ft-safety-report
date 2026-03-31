# `InternedStringsTable` (`kPyInternedStringToDimname`) concurrent hash map access

- **Status:** Open
- **Severity:** SEVERE
- **Component:** python_dimname.cpp

- **Shared state:** `kPyInternedStringToDimname` -- a static
  `InternedStringsTable` containing a `ska::flat_hash_map<PyObject*, at::Dimname>`
  (python_dimname.cpp:24). This is a process-global hash map.
- **Writer(s):**
  - `THPDimname_parse()` calls `kPyInternedStringToDimname.addMapping(obj, dimname)`
    when a new dimname string is encountered (line 107). `addMapping` calls
    `py_interned_string_to_dimname_.emplace()` which may rehash the map.
    This is called from any Python thread that constructs or manipulates
    named tensors.
- **Reader(s):**
  - `THPDimname_parse()` calls `kPyInternedStringToDimname.lookup(obj)` (line
    100), which calls `py_interned_string_to_dimname_.find()`. Called from any
    Python thread using named tensors.
  - The destructor iterates the map to `Py_DECREF` all keys.
- **Race scenario:** Thread A calls `THPDimname_parse("batch")` for the first
  time and enters `addMapping`, which calls `emplace()` on the flat hash map.
  Thread B concurrently calls `THPDimname_parse("feature")` and enters
  `lookup`, which calls `find()` on the same map. The `ska::flat_hash_map` is
  an open-addressing hash map -- `emplace` may trigger a rehash that
  reallocates the internal storage array. Thread B's `find()` is iterating
  through the old (now-freed) storage. This is a use-after-free / container
  corruption crash.
  Even without rehash, concurrent insert + find on a flat hash map is
  undefined behavior because insert modifies the control bytes that find reads.
- **Suggested fix:** Protect the map with a `std::mutex` or
  `std::shared_mutex`. The dimname parsing path is not performance-critical
  (named tensors are a relatively niche feature), so a simple mutex suffices.
  Alternatively, since interned Python strings have stable identity, a
  lock-free concurrent map could be used, but a mutex is simpler.
