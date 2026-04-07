# `c_recursion_limit` non-atomic int32_t

- **Status:** Not applicable
- **Severity:** N/A
- **Tier:** N/A
- **Component:** eval_frame

`CRecursionLimitRAII`, the only in-C++ reader of `c_recursion_limit`, is
compiled under `#if IS_PYTHON_3_12_PLUS && !IS_PYTHON_3_14_PLUS`
(eval_frame_cpp.cpp line 305). On 3.14+, the struct is a no-op. Since
free-threading only applies to 3.14t+, this issue is not relevant.

The getter/setter (`dynamo_get_c_recursion_limit` /
`dynamo_set_c_recursion_limit`) still exist but are unused on 3.14t.
