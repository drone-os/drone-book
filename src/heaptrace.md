# Heap Tracing

Drone OS provide tools to fine-tune the built-in allocator for purposes of a
particular application. A newly generated Drone project has a predefined
`heaptrace` feature. It is used in the `heap!` macro inside `src/lib.rs`:

```rust
heap! {
    /// A heap allocator generated from the `Drone.toml`.
    pub struct Heap;

    #[cfg(feature = "heaptrace")] use drone_cortex_m::itm::trace_alloc;
    #[cfg(feature = "heaptrace")] use drone_cortex_m::itm::trace_dealloc;
    #[cfg(feature = "heaptrace")] use drone_cortex_m::itm::trace_grow_in_place;
    #[cfg(feature = "heaptrace")] use drone_cortex_m::itm::trace_shrink_in_place;
}
```

When the feature is activated, special hooks are attached to the generated heap
code, which will log the allocator operations to the ITM port #31. In order to
capture these logs, a version of the application firmware with this feature
activated needs to be flashed to the target device first:

```shell
$ just features=heaptrace flash
```

Then you run a special recipe to capture the data:

```shell
$ just heaptrace
```

This recipe is similar to `just itm`, with an exception that the data from the
ITM port #31 will be written to the `heaptrace` file. When you think it is
enough data collected, just stop it with Ctrl-C.

The `heaptrace` feature doesn't add much of additional code to the binary. But
it may slow down the execution of your program in allocation-heavy scenarios
considerably. Because it must wait until the log data is transmitted over the
ITM port. Though this run-time overhead applies only when a debug probe is
attached, and the `just heaptrace` command is listening.

When there is a non-empty `heaptrace` file with the collected data in the
project root, you may use the following command to analyze your heap usage:

```shell
$ drone heap
```

It will print statistics of all your allocations during `just heaptrace`:

```text
---------------------------------- HEAP USAGE ----------------------------------
 <size> <max count> <allocations>
      1           1             1
     12           3             7
     28           1             2
     32           1             1
    128           1             2
Maximum memory usage: 225 / 2.20%
```

The data in the `heaptrace` file can also be used to generate an optimized
memory pools layout:

```shell
$ drone heap generate --pools 5
```

Here `5` is the maximum number of pools. Less pools lead to more fragmentation,
but faster allocations. You should get something like this:

```text
------------------------------- SUGGESTED LAYOUT -------------------------------
# Fragmentation: 0 / 0.00%

[heap]
size = "10K"
pools = [
    { block = "4", capacity = 201 },
    { block = "12", capacity = 222 },
    { block = "28", capacity = 115 },
    { block = "32", capacity = 83 },
    { block = "128", capacity = 7 },
]
```

It generated a `[heap]` section suitable to put into the `Drone.toml`.
