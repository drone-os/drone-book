# Drone OS Extensibility

Drone is designed to be maximally extensible to various platforms. It is
composed of a complex hierarchy of Rust crates. Where the main foundational part
if fully platform-agnostic, and platform-specific crates are built on top of it.

The core part of Drone makes little to no assumptions about the platform it will
be running on. One exception is that the platform should have a good support of
atomic operations on the instruction level. Drone tries hard to never use
disabling of interrupts to protect its shared data-structures.

In this section we will review Drone crates hierarchy by the example of Nordic
Semiconductor's nRF91 microcontroller series.

![Crates Hierarchy](../assets/crates-hierarchy.svg)

The crates composed of the following workspaces:

* **drone** - Drone command-line utility.

* **drone-core** - Drone core functionality.

* **drone-cortexm** - ARM® Cortex®-M support.

* **drone-svd** - CMSIS-SVD file format parser.

* **drone-nrf-map** - Nordic Semiconductor nRFx mappings.

* **drone-nrf91-dso** - Drone Serial Output implementation for Nordic
  Semiconductor nRF91.

## Adding New Chip Support

In order to add Drone support for a not-yet-supported chip, firstly we need to
determine its platform. If the platform is not yet supported, e.g. RISC-V, we
start by creating `drone-riscv` crate. If the platform is already supported,
e.g. Cortex-M, but the platform version is not yet supported, e.g. Cortex-M23,
we extend the existing `drone-cortexm` crate.

When we have the platform support, we need to add registers and interrupt
mappings. We need to find out if there is already a crate for the chip
series. If there is no such crate, e.g. Texas Instruments SimpleLink™, we need
to create one: `drone-tisl-map`. If, for example, we need to add support for
STM32WB55, we need to extend the existing `drone-stm32-map` crate.

If the chip doesn't have hardware logging capabilities (e.g. SWO), we need to
write a crate, which implements DSO (Drone Serial Output) protocol in
software. By, for example, using generic UART peripheral.

Lastly, we need to let the `drone` CLI utility to know about the chip. There
should be at least one debugger and at least one logger options for the
chip. This will be covered in the next section.
