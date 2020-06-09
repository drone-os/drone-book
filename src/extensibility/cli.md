# Drone CLI

In terms of chip support, Drone CLI is responsible for the following:

1. Generating a correct scaffold for a new project. The generated program should
   be ready to flash into the chip. The program should print `"Hello, world!"`
   string to the standard output.

2. Generating a correct linker script.

3. Working with the chip through one or more debug probes.

4. At least one method of capturing Drone logger output.

All platform-specific crates should be registered at `drone/src/crates.rs`. This
includes platform crates (e.g. `drone-cortexm`), vendor-specific mappings
(e.g. `drone-stm32-map`, `drone-nrf-map`), and DSO (Drone Serial Output )
implementation crates (e.g. `drone-nrf91-dso`.)

Specific microcontroller models should be registered at
`drone/src/devices/registry.rs`. For example here is an entry for Nordic
Semiconductor nRF9160:

```rust
    Device {
        name: "nrf9160", // device identifier
        target: "thumbv8m.main-none-eabihf", // Rust target triple
        flash_origin: 0x0000_0000, // Starting address of Flash memory
        ram_origin: 0x2000_0000, // Starting address of RAM
        // A link to the platform crate with specific flags and features
        platform_crate: PlatformCrate {
            krate: crates::Platform::Cortexm,
            flag: "cortexm33f_r0p2",
            features: &[
                "floating-point-unit",
                "memory-protection-unit",
                "security-extension",
            ],
        },
        // A link to the bindings crate with specific flags and features
        bindings_crate: BindingsCrate {
            krate: crates::Bindings::Nrf,
            flag: "nrf9160",
            features: &["uarte"],
        },
        probe_bmp: None, // BMP is unsupported
        probe_openocd: None, // OpenOCD is unsupported
        probe_jlink: Some(ProbeJlink { device: "NRF9160" }), // J-Link configuration
        log_swo: None, // SWO is unsupported
        // A link to the DSO implementation
        log_dso: Some(LogDso { krate: crates::Dso::Nrf91, features: &[] }),
    },
```

`drone` CLI provides a unified interface to various debug probes. There are
currently three supported types of debug probes: Black Magic Probe, J-Link, and
OpenOCD, which is itself an interface to different debuggers. In order to add a
new chip support to Drone, the CLI utility should be taught how to use the chip
through one of the currently known probes, or a completely new probe support can
be added for this chip.

The CLI utility is also responsible for capturing data from built-in Drone
logger. There are currently two protocol parsers implemented: SWO (ARM's Serial
Wire Output) and DSO (Drone Serial Output.) DSO protocol is used when there is
no hardware protocol implemented on the chip. The log output can be captured
through probe's built-in reader, or through generic external UART reader. At
least one log method should be implemented for a new chip.
