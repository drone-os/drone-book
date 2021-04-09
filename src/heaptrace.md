# Heap Tracing

Drone OS provides tools to fine-tune the built-in allocator for purposes of a
particular application.

A newly generated Drone project has the following `heap!` macro in `src/lib.rs`:

```rust
heap! {
    // Heap configuration key in `Drone.toml`.
    config => main;
    /// The main heap allocator generated from the `Drone.toml`.
    metadata => pub Heap;
    // Use this heap as the global allocator.
    global => true;
    // Uncomment the following line to enable heap tracing feature:
    // trace_port => 31;
}
```

Note that `trace_port` option is commented out - by default the firmware
compiles without the heap tracing runtime. When the option is uncommented, the
heap allocator will log its operations to the log port #31. In order to capture
these logs, first uncomment the `trace_port` option:

```rust
heap! {
    // ... The header is skipped ...

    // Uncomment the following line to enable heap tracing feature:
    trace_port => 31;
}
```

Then flash the new version of the application firmware to the target device:

```shell
$ just flash
```

Then you run a special recipe to capture the data:

```shell
$ just heaptrace
```

This recipe is similar to `just log`, with an exception that it will
additionally capture port #31 output and write it to a file named
`heaptrace`. When you think it is enough data collected, just stop it with
Ctrl-C.

When there is a non-empty `heaptrace` file with the collected data in the
project root, you may use the following command to analyze your heap usage:

```shell
$ drone heap
```

It will print statistics of all your allocations during `just heaptrace`:

```text
 Block Size | Max Load | Total Allocations
------------+----------+-------------------
          1 |        1 |                 1
         12 |        3 |                 7
         28 |        1 |                 2
         32 |        1 |                 1
        128 |        1 |                 2

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
=============================== SUGGESTED LAYOUT ===============================
[heap]
size = "10K"
pools = [
    { block = "4", capacity = 201 },
    { block = "12", capacity = 222 },
    { block = "28", capacity = 115 },
    { block = "32", capacity = 83 },
    { block = "128", capacity = 7 },
]
# Fragmentation: 0 / 0.00%
```

It generated a `[heap]` section suitable to put into the `Drone.toml`.
