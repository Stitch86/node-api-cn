<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```C
NAPI_EXTERN napi_status napi_throw_range_error(napi_env env,
                                               const char* code,
                                               const char* msg);
```

- `[in] env`: The environment that the API is invoked under.
- `[in] code`: Optional error code to be set on the error.
- `[in] msg`: C string representing the text to be associated with
the error.

Returns `napi_ok` if the API succeeded.

This API throws a JavaScript `RangeError` with the text provided.

