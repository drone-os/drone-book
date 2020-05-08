# Peripheral Mappings

Peripheral mappings serves two main purposes: grouping memory-mapped registers
and individual register fields together in a single block for convenient use,
and making one generic block for multiple peripherals of the same type
(e.g. SPI1, SPI2, SPI3).

While register mappings we are able to generate almost automatically from SVD
files (they are often of poor quality, and require manual fix-ups), we define
peripheral mappings manually for each supported target with help of powerful
procedure macros. For this reason we can't map all available peripherals for all
targets, but we strive the mapping process to be as easy as possible. So users
could map missing peripherals by themselves, and maybe contribute it back to
Drone OS. For the details how to create peripheral mappings, refer to the
`drone_core::periph` documentation.

A peripheral mapping defines a macro to acquire all needed register tokens. In
the following example, `periph_gpio_c!` and `periph_sys_tick!` are such macros:

```rust
pub fn handler(reg: Regs, thr_init: ThrsInit) {
    let gpio_c = periph_gpio_c!(reg);
    let sys_tick = periph_sys_tick!(reg);
    beacon(gpio_c, sys_tick)
}
```

`gpio_c` and `sys_tick` objects are zero-sized, and these lines of code incur no
run-time cost. These objects hold all relevant register and field tokens for the
corresponding peripherals. It is impossible to create two instances for a single
peripheral, because after the first macro call the `reg` object becomes
partially-moved.

The `beacon` function could be defined as follows:

```rust
fn beacon(
    gpio_c: GpioPortPeriph<GpioC>,
    sys_tick: SysTickPeriph,
) {
    // ...
}
```

Note that the type of `gpio_c` argument is a generic struct, because there are
many possible peripherals with the same interface: `GpioA`, `GpioB`, `GpioC`,
and so on. Conversely, the `sys_tick` type is not generic, because there is only
one SysTick peripheral in the chip. We could easily define the `beacon` function
to be generic over GPIO port:

```rust
fn beacon<GpioPort: GpioPortMap>(
    gpio_c: GpioPortPeriph<GpioPort>,
    sys_tick: SysTickPeriph,
) {
    // ...
}
```

This is a preferred and very handy way to define drivers. We don't want to
hard-code an SD-card driver to use for example only SPI3. An alternative
approach would be to wrap the whole driver code into a macro, and call it with
SPI1, SPI2, SPI3 arguments. But we believe this would be a less clean and
idiomatic way.
