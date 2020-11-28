# Memory-Mapped Registers

Modern processor architectures (e.g. ARM) use memory-mapped I/O to perform
communication between the CPU and peripherals. Using memory-mapped registers is
a considerable part in programming for microcontroller. Therefore Drone OS
provides a complex API, which provides convenient access to them without
data-races.

For example in STM32F103, the memory address of `0x4001100C` corresponds to the
GPIOC_ODR register. This is a register to control the output state of the GPIO
port C peripheral.

```rust
use core::ptr::write_volatile;

unsafe {
    write_volatile(0x4001_100C as *mut u32, 1 << 13);
}
```

The above code is an example how to write to a memory-mapped register in bare
Rust, without Drone. It sets PC13 pin output to logic-high (resetting all other
port C pins to logic-low.) This code is too low-level and error-prone, and also
requires an `unsafe` block.

For Cortex-M there is SVD (System View Description) format. Vendors generally
provide files of this format for their Cortex-M MCUs. Drone generates
MCU-specific register API from these files for each supported target. So copying
addresses and offsets from reference manuals generally is not needed.

Let's look at the default `reset` function in `src/bin/<crate-name>.rs`, which
is the entry-point of the program:

```rust
#[no_mangle]
#[naked]
pub unsafe extern "C" fn reset() -> ! {
    mem::bss_init();
    mem::data_init();
    tasks::root(Regs::take(), ThrsInit::take());
    loop {
        processor::wait_for_int();
    }
}
```

This `unsafe` function performs all necessary initialization routines before
calling the safe `root` entry task. This includes `Regs::take()` and
`ThrsInit::take()` calls. These calls create instances of `Regs` and `ThrsInit`
types, which are zero-sized types. The calls are `unsafe`, because they must be
done only once in the whole program run-time.

Let's now check the `tasks::root` function (it is re-exported from `handler`):

```rust
pub fn handler(reg: Regs, thr_init: ThrsInit) {
    // Enter a sleep state on ISR exit.
    reg.scb_scr.sleeponexit.set_bit();
}
```

`reg` is an open-struct (all fields of the struct are `pub`) and consists of all
available register tokens. Each register token is also an open-struct and
consists of register field tokens. So this line:

```rust
    reg.scb_scr.sleeponexit.set_bit();
```

Sets SLEEPONEXIT bit of SCB_SCR register.

Of course no real-world application would use all available memory-mapped
registers. The `reg` object is supposed to be destructured within the `root`
task handler and automatically dropped. To make this more readable, we move
individual tokens out of `reg` in logical blocks using macros:

```rust
pub fn handler(reg: Regs, thr_init: ThrsInit) {
    let gpio_c = periph_gpio_c!(reg);
    let sys_tick = periph_sys_tick!(reg);
    beacon(gpio_c, sys_tick)
}
```

These macros use partial-moving feature of Rust and expand roughly as follows:

```rust
pub fn handler(reg: Regs, thr_init: ThrsInit) {
    let gpio_c = GpioC {
        gpio_crl: reg.gpio_crl,
        gpio_crh: reg.gpio_crh,
        gpio_idr: reg.gpio_idr,
        gpio_odr: reg.gpio_odr,
        // Notice that below are individual fields.
        // Other APB2 peripherals may take other fields from this same registers.
        rcc_apb2enr_iopcen: reg.rcc_apb2enr.iopcen,
        rcc_apb2enr_iopcrst: reg.rcc_apb2enr.iopcrst,
        // ...
    };
    let sys_tick = SysTick {
        stk_ctrl: reg.stk_ctrl,
        stk_load: reg.stk_load,
        stk_val: reg.stk_val,
        scb_icsr_pendstclr: reg.scb_icsr.pendstclr,
        scb_icsr_pendstset: reg.scb_icsr.pendstset,
    };
    beacon(gpio_c, sys_tick)
}
```

If you wonder why we use macros instead of functions, the following example
shows why functions wouldn't work:

```rust
fn periph_gpio_c(reg: Regs) -> GpioC {
    GpioC {
        gpio_crl: reg.gpio_crl,
        gpio_crh: reg.gpio_crh,
        gpio_idr: reg.gpio_idr,
        gpio_odr: reg.gpio_odr,
        // Notice that below are individual fields.
        // Other APB2 peripherals may take other fields from this same registers.
        rcc_apb2enr_iopcen: reg.rcc_apb2enr.iopcen,
        rcc_apb2enr_iopcrst: reg.rcc_apb2enr.iopcrst,
        // ...
    }
}

fn periph_sys_tick(reg: Regs) -> GpioC {
    SysTick {
        stk_ctrl: reg.stk_ctrl,
        stk_load: reg.stk_load,
        stk_val: reg.stk_val,
        scb_icsr_pendstclr: reg.scb_icsr.pendstclr,
        scb_icsr_pendstset: reg.scb_icsr.pendstset,
    }
}

pub fn handler(reg: Regs, thr_init: ThrsInit) {
            // --- move occurs because `reg` has type `Regs`, which
            //     does not implement the `Copy` trait
    let gpio_c = periph_gpio_c!(reg);
                             // --- value moved here
    let sys_tick = periph_sys_tick!(reg);
                                 // --- value used here after move
    beacon(gpio_c, sys_tick)
}
```
