# ld.so Interface

The following is the basic interface specification for the binary interface exposed by `ld-lilium.so`. Note that the following is an ABI Specification and has no source prototypes in any language.

## TLS Management

### `__rtld_update_global_tcb`

Prototype:

```rust
unsafe extern "system" {
    fn __rtld_update_global_tcb();
}
```

Behaviour:
* After a call to this function on a thread, all dynamically allocated tls variables and all statically allocated tls variables loaded prior to the call to `__rtld_update_global_tcb()`, are made available in the calling thread.
* The function *synchronizes-with* Any call to `__rtld_open_module` that loads new tls variables.