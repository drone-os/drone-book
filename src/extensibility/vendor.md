# Vendor-Specific Layer

This layer consists of memory-mapped register, interrupt, and peripheral
mappings, and also possibly of DSO (Drone Serial Output) implementations. In
this section we will overview `drone-nrf-map` and `drone-nrf91-dso` crates as
examples of the vendor-specific layer.

## Bindings

`drone-nrf-map` collection of crates is purely declarative. We try to
automatically generate as much code as possible from vendor-provided CMSIS-SVD
files. Generation of memory-mapped register bindings is highly
parallelizable. Therefore it's splitted into 12 crates, which are named from
`drone-nrf-map-pieces-1` to `drone-nrf-map-pieces-12` and compiled by cargo in
parallel. `drone-nrf-map-pieces-*` crates are all re-exported by single
`drone-nrf-map-pieces` crate, which can be further used by peripheral bindings.

Not all bindings can be auto-generated. We also manually declare [peripheral
mappings](../periph.md). For the sake of compile-time parallelization, each
peripheral type is declared in its own crate
(e.g. `drone-nrf-map-periph-uarte`.) Periheral crates are opt-in, they are
enabled by activating corresponding cargo features for `drone-nrf-map` crate.

Finally, `drone-nrf-map-pieces` and `drone-nrf-map-periph-*` crates are all
re-exported by `drone-nrf-map` crate.

## Drone Serial Output

If the target doesn't implement usual hardware logging, as in case with nRF9160,
we provide a software logging implementation. It uses special Drone Serial
Output protocol to provide features similar to hardware SWO. Namely splitting
the output into different ports and forming atomic packets.

`drone-nrf91-dso` implementation is based on software output FIFO, and utilizes
one of generic built-in UART peripheral.
