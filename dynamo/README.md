# Dynamo Free-Threading Issues

Issues in `torch/csrc/dynamo/` affecting free-threaded Python 3.14t.

## Tier 1 (urgent: single-thread compile + data loaders)

| Severity | Component | Issue |
|----------|-----------|-------|
| SEVERE | compiled_autograd | [`CacheNode` tree: `clear_cache` races with backward traversal](cachenode-tree-clear-cache-races-with-backward-traversal.md) |
| SEVERE | compiled_autograd | [`python_verbose_logger` borrowed reference: use-after-free](python-verbose-logger-borrowed-reference-use-after-free.md) |
| SEVERE | compiled_autograd | [`the_autograd_compiler` pointer: unsynchronized read vs. write](the-autograd-compiler-pointer-unsynchronized-read-vs-write.md) |
| SEVERE | eval_frame | [Concurrent access to `cache_entry_list` (std::list) during lookup, ins](concurrent-access-to-cache-entry-list-std-list-during-lookup-insertion-invalidat.md) |
| SEVERE | guards | [`dict_recursive_tag_watch_callback` calls `disable_recursive_dict_tag_](dict-recursive-tag-watch-callback-calls-disable-recursive-dict-tag-optimization-.md) |
| SEVERE | guards | [`dict_to_guard_managers` concurrent access from watcher callbacks and ](dict-to-guard-managers-concurrent-access-from-watcher-callbacks-and-guard-evalua.md) |
| SEVERE | guards | [`dict_version_map` and `global_dict_version_id` concurrent access from](dict-version-map-and-global-dict-version-id-concurrent-access-from-watcher-callb.md) |
| SEVERE | guards | [`disable_dict_tag_matching_callback` (weakref callback) dereferences `](disable-dict-tag-matching-callback-weakref-callback-dereferences-guardmanager-co.md) |
| Significant | compiled_autograd | [`default_dyn_type_int` non-atomic int](default-dyn-type-int-non-atomic-int.md) |
| Minor | compiled_autograd | [`Py_MOD_GIL_NOT_USED` is the root cause of S1-S4](py-mod-gil-not-used-is-the-root-cause-of-s1-s4.md) |
| Minor | guards | [`global_dict_version_id` non-atomic increment](global-dict-version-id-non-atomic-increment.md) |

## Tier 2 (goal: full multi-thread torch.compile)

| Severity | Component | Issue |
|----------|-----------|-------|
| SEVERE | compiled_autograd | [`active_rstate` global pointer: unsynchronized read from `call_cpp_ten](active-rstate-global-pointer-unsynchronized-read-from-call-cpp-tensor-pre-hooks.md) |
| SEVERE | eval_frame | [`breakpoint_code_objects` (std::unordered_set) concurrent access](breakpoint-code-objects-std-unordered-set-concurrent-access.md) |
| SEVERE | eval_frame | [Concurrent access to `precompile_entries` (std::list)](concurrent-access-to-precompile-entries-std-list.md) |
| SEVERE | eval_frame | [`init_and_set_extra_state` check-then-act TOCTOU](init-and-set-extra-state-check-then-act-toctou.md) |
| Significant | compiled_autograd | [`CacheNode::check_dynamic_sizes` mutates shared state without external](cachenode-check-dynamic-sizes-mutates-shared-state-without-external-protection.md) |
| Significant | compiled_autograd | [`kActivePyCompilerInterface` unprotected global `unique_ptr`](kactivepycompilerinterface-unprotected-global-unique-ptr.md) |
| Significant | compiled_autograd | [`set_autograd_compiler` non-atomic read-modify-write sequence](set-autograd-compiler-non-atomic-read-modify-write-sequence.md) |
| Significant | eval_frame | [`active_dynamo_threads` counter race](active-dynamo-threads-counter-race.md) |
| Significant | eval_frame | [`eval_frame_override` global variable non-atomic access](eval-frame-override-global-variable-non-atomic-access.md) |
| Significant | eval_frame | [`ExtraState::frame_state` (py::dict) concurrent access](extrastate-frame-state-py-dict-concurrent-access.md) |
| Significant | eval_frame | [`ExtraState::strategy` non-atomic read/write](extrastate-strategy-non-atomic-read-write.md) |
| Significant | eval_frame | [`guard_error_hook` and `guard_complete_hook` global PyObject* pointers](guard-error-hook-and-guard-complete-hook-global-pyobject-pointers.md) |
| Significant | eval_frame | [`previous_eval_frame` function pointer race in enable/disable](previous-eval-frame-function-pointer-race-in-enable-disable.md) |
| Significant | guards | [`_accessors` sort and `_fail_count` mutation unprotected when `check` ](accessors-sort-and-fail-count-mutation-unprotected-when-check-is-called-from-pyt.md) |
| Significant | guards | [`GuardManager::_dict_tag` updated without synchronization across roots](guardmanager-dict-tag-updated-without-synchronization-across-roots-watching-the-.md) |
| Significant | guards | [Shared `RelationalGuard` instances across cloned `RootGuardManager`s r](shared-relationalguard-instances-across-cloned-rootguardmanagers-race-on-mutable.md) |
| Minor | compiled_autograd | [`CompiledAutogradThreadingDebugCheck` TOCTOU in engine.cpp](compiledautogradthreadingdebugcheck-toctou-in-engine-cpp.md) |
| Minor | compiled_autograd | [Static local `PyObject*` from `PyUnicode_InternFromString` in hot path](static-local-pyobject-from-pyunicode-internfromstring-in-hot-path.md) |
| Minor | eval_frame | [`bytecode_debugger_callback_obj` and `random_module` global pointers](bytecode-debugger-callback-obj-and-random-module-global-pointers.md) |
| Minor | eval_frame | [`c_recursion_limit` non-atomic int32_t](c-recursion-limit-non-atomic-int32-t.md) |
| Minor | eval_frame | [`convert_frame_get_fail_callback` static optional lazy init](convert-frame-get-fail-callback-static-optional-lazy-init.md) |
| Minor | eval_frame | [`is_skip_guard_eval_unsafe` non-atomic bool](is-skip-guard-eval-unsafe-non-atomic-bool.md) |
| Minor | eval_frame | [`use_lru` non-atomic bool](use-lru-non-atomic-bool.md) |
| Minor | guards | [`make_guard_manager` static local initialization on pre-pybind 2.13](make-guard-manager-static-local-initialization-on-pre-pybind-2-13.md) |
| Minor | guards | [`StorageOverlapChecker` reference counting races when shared across cl](storageoverlapchecker-reference-counting-races-when-shared-across-cloned-roots.md) |

## Unclassified

- [`CacheNode` destructor leak on interpreter shutdown](cachenode-destructor-leak-on-interpreter-shutdown.md)

