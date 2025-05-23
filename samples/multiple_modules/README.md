# Multiple Modules / Pipelines

IREE programs can be composed of multiple modules and calls can be made between
them even if the modules are implemented manually (C/C++/python/etc) or
generated by the compiler (VMFB/emitc). Modules have a user-specified load
order and can only reference modules loaded prior. Each module can contain
global state that is scoped to the VM context that loaded them to allow for
multiple copies of a program to be loaded in a single hosting process.

This sample contains two modules produced by the compiler
([`module_a.mlir`](./module_a.mlir) and [`module_b.mlir`](./module_b.mlir)) that
export some basic functions and a pipeline module implemented for both
synchronous ([`pipeline_sync.mlir`](./pipeline_sync.mlir)) and asynchronous
execution ([`pipeline_async.mlir`](./pipeline_async.mlir)) that calls into the
modules. This shows how common/reusable portions of programs can be factored
into multiple components and complex stateful pipelines can be easily authored
that use those components.

## Instructions (synchronous)

1. Compile each module to .vmfb files:

    ```
    iree-compile \
        --iree-hal-target-device=local \
        --iree-hal-local-target-device-backends=vmvx \
        module_a.mlir -o=module_a_sync.vmfb
    iree-compile \
        --iree-hal-target-device=local \
        --iree-hal-local-target-device-backends=vmvx \
        module_b.mlir -o=module_b_sync.vmfb
    iree-compile \
        --iree-hal-target-device=local \
        --iree-hal-local-target-device-backends=vmvx \
        pipeline_sync.mlir -o=pipeline_sync.vmfb
    ```

2. Run the program by passing in the modules in load order (dependencies first):

    ```
    iree-run-module \
        --device=local-sync \
        --module=module_a_sync.vmfb \
        --module=module_b_sync.vmfb \
        --module=pipeline_sync.vmfb \
        --function=run \
        --input=4096xf32=-2.0
    ```

## Instructions (asynchronous)

The compilation portion is the same as above just with the addition of the flag
telling the compiler to make the modules export functions with async support.
It's possible to have modules that contain a mix of synchronous and asynchronous
exports and calls if desired. `iree-run-module` and other IREE tooling
automatically detects asynchronous calls and handles adding the wait and signal
fences while applications directly using IREE APIs for execution will need to
add the fences themselves.

1. Compile each module to .vmfb files with the `async-external` ABI model:

    ```
    iree-compile \
        --iree-execution-model=async-external \
        --iree-hal-target-device=local \
        --iree-hal-local-target-device-backends=vmvx \
        module_a.mlir -o=module_a_async.vmfb
    iree-compile \
        --iree-execution-model=async-external \
        --iree-hal-target-device=local \
        --iree-hal-local-target-device-backends=vmvx \
        module_b.mlir -o=module_b_async.vmfb
    iree-compile \
        --iree-execution-model=async-external \
        --iree-hal-target-device=local \
        --iree-hal-local-target-device-backends=vmvx \
        pipeline_async.mlir -o=pipeline_async.vmfb
    ```

2. Run the program by passing in the modules in load order (dependencies first):

    ```
    iree-run-module \
        --device=local-sync \
        --module=module_a_async.vmfb \
        --module=module_b_async.vmfb \
        --module=pipeline_async.vmfb \
        --function=run \
        --input=4096xf32=-2.0
    ```
