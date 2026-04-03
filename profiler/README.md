# Profiler Free-Threading Issues

Issues in the PyTorch profiler subsystem affecting free-threaded Python 3.14t.

Files audited:
- `torch/csrc/autograd/profiler_python.cpp`
- `torch/csrc/autograd/profiler_kineto.cpp`
- `torch/csrc/profiler/python/combined_traceback.cpp`
- `torch/csrc/profiler/python/init.cpp`
- `torch/csrc/profiler/orchestration/python_tracer.cpp`
- `torch/csrc/profiler/collection.cpp`

## Issues

| Severity | Component | Issue |
|----------|-----------|-------|
| SEVERE | profiler_python, profiler_kineto, collection | [`pyProfileFn` teardown race: worker callbacks race with `disableProfiler`](pyprofilefn-teardown-race-with-worker-threads.md) ⏳ [#179262](https://github.com/pytorch/pytorch/pull/179262) |
| SEVERE | profiler_kineto | [`profiler_state_info_ptr` (shared_ptr) data race across threads](profiler-state-info-ptr-shared-ptr-data-race.md) |
| SEVERE | profiler_python | [GC callback use-after-free of `PythonTracer` instance](gc-callback-use-after-free-of-pythontracer.md) |
| Significant | profiler_kineto | [`toggleTorchOpCollectionDynamic` dereferences `profiler_state_info_ptr` wit](toggle-torch-op-collection-dynamic-null-deref.md) |
| Significant | profiler_python | [`py_gc_callback` global pointer race between registration and GC](py-gc-callback-global-pointer-race.md) |
| Significant | orchestration | [Global function pointers for tracer registration are non-atomic](global-function-pointers-for-tracer-registration.md) |
| Significant | collection | [Global `std::function` feature flags data race](global-std-function-feature-flags-data-race.md) |
| Minor | profiler_python | [`getCode<>` stores borrowed `PyCodeObject*` references](profiler-getcode-borrowed-ref.md) |
| Minor | profiler_python | [`CodeLocation` stores borrowed `const char*` from code objects](profiler-code-location-borrowed-ptrs.md) |
| Minor | profiler_python | [`thread_local_results_map_` fragile read-only-after-init design](profiler-thread-local-results-map-race.md) |
| Minor | combined_traceback | [`PyList_GetItem` borrowed reference in `gatherForwardTraceback`](pylist-getitem-borrowed-ref-in-gather-forward-traceback.md) |

<details>
<summary>Fixed (2 issues)</summary>

| Severity | Component | Issue | PR |
|----------|-----------|-------|----|
| SEVERE | profiler_python | [`PyThreadState_Swap` into running threads during profiler init](pythread-state-swap-into-running-threads.md) | [#178551](https://github.com/pytorch/pytorch/pull/178551) |
| SEVERE | profiler_python | [Shared `ValueCache` across threads](shared-valuecache-across-threads.md) | [#178552](https://github.com/pytorch/pytorch/pull/178552) |

</details>
