<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```C
napi_status napi_coerce_to_string(napi_env env,
                                  napi_value value,
                                  napi_value* result)
```

- `[in] env`: The environment that the API is invoked under.
- `[in] value`: The JavaScript value to coerce.
- `[out] result`: `napi_value` representing the coerced JavaScript `String`.

Returns `napi_ok` if the API succeeded.

This API implements the abstract operation `ToString()` as defined in

of the ECMAScript Language Specification.
This API can be re-entrant if getters are defined on the passed-in `Object`.

