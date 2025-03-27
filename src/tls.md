# Thread Local Storage

Lilium supports the ELF Thread Local Storage Specification. With the following notes below.

## Initial Exec

Under the base ELF TLS Specification, initial-exec model is defined only for the root executable, and shared objects (potentially indirectly) NEEDED from that executable.

Subject to the below, this is guaranteed to be supported for any shared object. 

To support Initial Exec, each thread that enters code of a post-loaded shared object that accesses thread-local storage (loaded by `__rtld_open_module`) must do the following:
* Call the function `__rtld_update_global_tcb` since the shared object was loaded, and
* The base executable must use the standard interpreter (Otherwise, the behaviour depends on the dynamic loader that executes the process).

Every thread spawned via the USI definition of `CreateThread` (in `libusi-thread`) guarantees the first property at all times, whenever `libusi-rtld.so` is dynamically linked. Note that if `libusi-thread.a` is statically linked by a given module, it is only guaranteed for calls to `CreateThread` from that module if either:
* The module dynamically links `libusi-rtld.so`, or
* The module is (potentially indirectly) NEEDED from the base executable and the base executable dynamically links `libusi-rtld.so`. 

If `libusi-thread.so` is dynamically linked by the module that calls `CreateThread`, it is sufficient that `libusi-rtld.so` is (potentially indirectly) NEEDED from the base executable, or by the module that makes the call (or any of its recursively NEEDED modules).

The following is guaranteed by the standard interpreter about when this function is called:
* It is called on any thread that calls `__rtld_open_module` immediately before that call returns, 
* It is called on any thread that calls `__tls_get_addr`, and
* It is called when that thread resolves a symbol in a post-loaded shared object via the plt or via `__rtld_lookup_module_sym`.

