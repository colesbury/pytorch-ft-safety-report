# `CacheNode` tree: `clear_cache` races with backward traversal

- **Status:** Open
- **Severity:** Minor
- **Tier:** Tier 1
- **Component:** compiled_autograd
- **Source report:** [dynamo_compiled_autograd_v2.md](../dynamo_compiled_autograd_v2.md)

- **Tier:** Tier 1 (technically reachable from any thread, but `clear_cache`
  / `reset` is primarily used in testing)
- **Shared state:** The entire `CacheNode` tree rooted at
  `CacheNode::root()`.
- **Writer(s):** `clear_cache()` (line 637) calls
  `CacheNode::root()->clear()`, which destroys all child nodes by clearing
  `next` (an `unordered_map` of `unique_ptr<CacheNode>`), `key_storage`,
  `expected_sizes`, and releases `runtime_wrapper`/`compiled_fn`.
- **Reader(s):** `_compiled_autograd_impl()` (line 949) traverses the tree via
  `cache = cache->lookup(key)`, obtaining raw pointers to child `CacheNode`s.
  Later (line 976) it calls `cache->check_dynamic_sizes()` and (lines
  1150-1157) reads `cache->runtime_wrapper` and `cache->compiled_fn`. Also,
  `compiled_autograd()` (lines 1238-1246) uses `cache->runtime_wrapper` and
  `cache->compiled_fn` after `_compiled_autograd_impl` returns.
- **Race scenario:** Thread A is inside `compiled_autograd()` holding the
  mutex, has traversed several cache nodes, and holds a raw pointer
  `cache` to a non-root `CacheNode`. Thread B calls `clear_cache()` from
  Python concurrently. `clear_cache` does not acquire the compiled autograd
  mutex. `CacheNode::root()->clear()` destroys all children via
  `next.clear()`, which invokes `unique_ptr<CacheNode>` destructors. Thread
  A's `cache` pointer is now dangling. Thread A proceeds to dereference it --
  use-after-free, likely a crash.
- **Suggested fix:** `clear_cache` must acquire the compiled autograd mutex
  before clearing.
