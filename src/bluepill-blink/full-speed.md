# Run at Full Speed

According to the datasheet, STM32F103 MCU can run at the maximum frequency of 72
MHz. But by default it runs at only 8 MHz. To achieve the full potential of the
chip, the system frequency should be raised in the run-time.

There are three options for the system clock source:

- HSI (High Speed Internal) - an RC oscillator running at constant 8 MHz and
  sitting inside the MCU chip. It is the default source for the system clock
  selected at the start-up.

- HSE (High Speed External) - an optional external resonator component in the
  range from 4 to 16 MHz. A Blue Pill board has a 8 MHz crystal connected to the
  MCU (the component in a metal case right beside the MCU marked as Y2.)

- PLL (Phase-Locked Loop) - a peripheral inside the MCU that can be used as a
  multiplier for HSI or HSE. The maximum multiplier for HSI is 8, which can give
  us 64 MHz, and for HSE - 16, which can theoretically result in 128 MHz, but
  the output frequency of PLL shouldn't exceed 72 MHz.

Given the above, in order to achieve 72 MHz, we should take the following steps:

1. Start the HSE oscillator and wait for it to stabilize.
2. Start the PLL with the HSE input and the multiplier of 9. Wait for it to
   stabilize.
3. Select the PLL as the source for the system clock.

For a start, let's create a module for our project-level constants. Create a new
file at `src/consts.rs` with the following content:

```rust
//! Project constants.

/// HSE crystal frequency.
pub const HSE_FREQ: u32 = 8_000_000;

/// PLL multiplication factor.
pub const PLL_MULT: u32 = 9;

/// System clock frequency.
pub const SYS_CLK: u32 = HSE_FREQ * PLL_MULT;
```

And register the module in the `src/lib.rs`:

```rust
pub mod consts;
```

When the application will need to wait for HSE and PLL clocks stabilization, we
don't want it to be constantly checking the flags wasting CPU cycles and energy,
but rather to subscribe for an interrupt and sleep until it is triggered. We
will use the RCC interrupt for this purpose:

![Vector Table](../assets/vtable-rcc.png)

From the table above, which can be found in the Reference Manual, we only need
the position of the RCC interrupt. Let's put this interrupt to the application
Vector Table. For this you need to edit `thr!` macro in `src/thr.rs`. By default
it looks like this:

```rust
thr! {
    // ... The header is skipped ...

    threads => {
        exceptions => {
            /// All classes of faults.
            pub hard_fault;
        };
    };
}
```

There is only a HardFault handler defined. Note that according the above table,
HardFault doesn't have a position number, therefore it is referred only by its
name. We need to add a new interrupt handler at the position of 5:

```rust
thr! {
    // ... The header is skipped ...

    threads => {
        exceptions => {
            /// All classes of faults.
            pub hard_fault;
        };
        interrupts => {
            /// RCC global interrupt.
            5: pub rcc;
        };
    };
}
```

Since the new handler has a numeric position, the name can be arbitrary.

Let's open the root task handler at `src/tasks/root.rs`. By default it looks
like this:

```rust
//! The root task.

use crate::{thr, thr::ThrsInit, Regs};
use drone_cortexm::{reg::prelude::*, thr::prelude::*};

/// The root task handler.
#[inline(never)]
pub fn handler(reg: Regs, thr_init: ThrsInit) {
    let thr = thr::init(thr_init);

    thr.hard_fault.add_once(|| panic!("Hard Fault"));

    println!("Hello, world!");

    // Enter a sleep state on ISR exit.
    reg.scb_scr.sleeponexit.set_bit();
}
```

In Drone OS the very first task with the lowest priority named root. Its
function handler is called by the program entry point at
`src/bin/bluepill-blink.rs`, after finishing unsafe initialization routines. The
root handler receives two arguments of types `Regs` and `ThrsInit`. Both are
zero-sized types implementing `Token` trait, which permits existence of only one
instance at a time. Instantiating a `Token` type is unsafe, that is why it is
done inside the unsafe entry point at `src/bin/bluepill-blink.rs`.

The `reg` argument is a set of tokens for all available memory-mapped
registers. It is supposed to be destructured into individual register tokens or
register field tokens within the root handler.

The second `thr_init` argument's purpose is to pass it to `thr::init`
function. The function runs an initialization routine for threading system and
returns an instance of `Thrs` type. `Thrs` is also a zero-sized type similar to
`Regs`, but for thread tokens.

The root handler adds a one-shot fiber to the HardFault thread. The fiber body
is just a call to the `panic!` macro. Drone handles panics by writing the panic
message to the log output at port #1, issuing a self-reset request, and blocking
until it's executed.

Let's add a new `async` function that will be responsible for raising the system
clock frequency to 72 MHz. It will need some registers from RCC and FLASH
peripherals, as well as the RCC thread token.

```rust
//! The root task.

use crate::{
    consts::{PLL_MULT, SYS_CLK},
    thr,
    thr::ThrsInit,
    Regs,
};
use drone_core::log;
use drone_cortexm::{fib, reg::prelude::*, swo, thr::prelude::*};
use drone_stm32_map::reg;

/// The root task handler.
#[inline(never)]
pub fn handler(reg: Regs, thr_init: ThrsInit) {
    let thr = thr::init(thr_init);

    thr.hard_fault.add_once(|| panic!("Hard Fault"));

    raise_system_frequency(
        reg.flash_acr,
        reg.rcc_cfgr,
        reg.rcc_cir,
        reg.rcc_cr,
        thr.rcc,
    )
    .root_wait();

    println!("Hello, world!");

    // Enter a sleep state on ISR exit.
    reg.scb_scr.sleeponexit.set_bit();
}

async fn raise_system_frequency(
    flash_acr: reg::flash::Acr<Srt>,
    rcc_cfgr: reg::rcc::Cfgr<Srt>,
    rcc_cir: reg::rcc::Cir<Srt>,
    rcc_cr: reg::rcc::Cr<Srt>,
    thr_rcc: thr::Rcc,
) {
    // TODO raise the frequency to 72 MHz
}
```

An `async` function is a syntax sugar for a function returning a custom type
implementing `Future` trait. We execute the returned future using the
`.root_wait()` method. The `root_wait` method is supposed to be used inside a
thread with the lowest priority, e.g. in the root task context. Otherwise the
threads that are currently preempted may be stalled. Another option for
executing futures is to use `exec` or `add_exec` methods on thread tokens.

It's good to check that the program still works:

```shell
$ just flash
$ just log
```

Let's start filling out the `raise_system_frequency` function. First, we need to
enable the RCC interrupt in NVIC (Nested Vectored Interrupt Controller), and
allow the RCC peripheral to trigger the interrupt when HSE or PLL is stabilized:

```rust
    thr_rcc.enable_int();
    rcc_cir.modify(|r| r.set_hserdyie().set_pllrdyie());
```

Then we're enabling the HSE clock and waiting until it's stabilized:

```rust
    // We need to move ownership of `hserdyc` and `hserdyf` into the fiber.
    let reg::rcc::Cir {
        hserdyc, hserdyf, ..
    } = rcc_cir;
    // Attach a listener that will notify us when RCC_CIR_HSERDYF is asserted.
    let hserdy = thr_rcc.add_future(fib::new_fn(move || {
        if hserdyf.read_bit() {
            hserdyc.set_bit();
            fib::Complete(())
        } else {
            fib::Yielded(())
        }
    }));
    // Enable the HSE clock.
    rcc_cr.modify(|r| r.set_hseon());
    // Sleep until RCC_CIR_HSERDYF is asserted.
    hserdy.await;
```

And similarly enable the PLL:

```rust
    // We need to move ownership of `pllrdyc` and `pllrdyf` into the fiber.
    let reg::rcc::Cir {
        pllrdyc, pllrdyf, ..
    } = rcc_cir;
    // Attach a listener that will notify us when RCC_CIR_PLLRDYF is asserted.
    let pllrdy = thr_rcc.add_future(fib::new_fn(move || {
        if pllrdyf.read_bit() {
            pllrdyc.set_bit();
            fib::Complete(())
        } else {
            fib::Yielded(())
        }
    }));
    rcc_cfgr.modify(|r| {
        r.set_pllsrc() // HSE oscillator clock selected as PLL input clock
            .write_pllmul(PLL_MULT - 2) // output frequency = input clock × PLL_MULT
    });
    // Enable the PLL.
    rcc_cr.modify(|r| r.set_pllon());
    // Sleep until RCC_CIR_PLLRDYF is asserted.
    pllrdy.await;
```

The flash memory settings should be tweaked because of increased frequency:

```rust
    // Two wait states, if 48 MHz < SYS_CLK <= 72 Mhz.
    flash_acr.modify(|r| r.write_latency(2));
```

Before increasing the frequency, we should wait until the currently ongoing SWO
transmission is finished if any. And also update the SWO prescaler to maintain
the fixed baud-rate defined at the project's `Drone.toml`. Note that if a debug
probe is not connected, this will be a no-op, thus it's safe to keep this in the
release binary.

```rust
    swo::flush();
    swo::update_prescaler(SYS_CLK / log::baud_rate!() - 1);
```

And finally switch the source for the system clock to PLL:

```rust
    rcc_cfgr.modify(|r| r.write_sw(0b10)); // PLL selected as system clock
```

Here is the final listing of the `raise_system_frequency` function:

```rust
async fn raise_system_frequency(
    flash_acr: reg::flash::Acr<Srt>,
    rcc_cfgr: reg::rcc::Cfgr<Srt>,
    rcc_cir: reg::rcc::Cir<Srt>,
    rcc_cr: reg::rcc::Cr<Srt>,
    thr_rcc: thr::Rcc,
) {
    thr_rcc.enable_int();
    rcc_cir.modify(|r| r.set_hserdyie().set_pllrdyie());

    // We need to move ownership of `hserdyc` and `hserdyf` into the fiber.
    let reg::rcc::Cir {
        hserdyc, hserdyf, ..
    } = rcc_cir;
    // Attach a listener that will notify us when RCC_CIR_HSERDYF is asserted.
    let hserdy = thr_rcc.add_future(fib::new_fn(move || {
        if hserdyf.read_bit() {
            hserdyc.set_bit();
            fib::Complete(())
        } else {
            fib::Yielded(())
        }
    }));
    // Enable the HSE clock.
    rcc_cr.modify(|r| r.set_hseon());
    // Sleep until RCC_CIR_HSERDYF is asserted.
    hserdy.await;

    // We need to move ownership of `pllrdyc` and `pllrdyf` into the fiber.
    let reg::rcc::Cir {
        pllrdyc, pllrdyf, ..
    } = rcc_cir;
    // Attach a listener that will notify us when RCC_CIR_PLLRDYF is asserted.
    let pllrdy = thr_rcc.add_future(fib::new_fn(move || {
        if pllrdyf.read_bit() {
            pllrdyc.set_bit();
            fib::Complete(())
        } else {
            fib::Yielded(())
        }
    }));
    rcc_cfgr.modify(|r| {
        r.set_pllsrc() // HSE oscillator clock selected as PLL input clock
            .write_pllmul(PLL_MULT - 2) // output frequency = input clock × PLL_MULT
    });
    // Enable the PLL.
    rcc_cr.modify(|r| r.set_pllon());
    // Sleep until RCC_CIR_PLLRDYF is asserted.
    pllrdy.await;

    // Two wait states, if 48 MHz < SYS_CLK <= 72 Mhz.
    flash_acr.modify(|r| r.write_latency(2));

    swo::flush();
    swo::update_prescaler(SYS_CLK / log::baud_rate!() - 1);

    rcc_cfgr.modify(|r| r.write_sw(0b10)); // PLL selected as system clock
}
```
