# Processes

Processes in Drone OS are special kind of fibers, that can be suspended with a
special blocking call. They use dedicated dynamically allocated stacks. On
Cortex-M platform, Drone implements processes using `SVC` assembly instruction
and SVCall exception. So before using processes, a Drone supervisor should be
added to the project.

## Supervisor

Create a new file at `src/sv.rs` with the following content:

```rust
//! The supervisor.

use drone_cortex_m::{
    sv,
    sv::{SwitchBackService, SwitchContextService},
};

sv! {
    /// The supervisor type.
    pub struct Sv;

    /// The array of services.
    static SERVICES;

    SwitchContextService;
    SwitchBackService;
}
```

And register the newly created module in the `src/lib.rs`:

```rust
pub mod sv;
```

Update `thr::vtable!` macro inside `src/thr.rs` as follows:

```rust
use crate::sv::Sv;

thr::vtable! {
    use Thr;
    use Sv; // <-- register the supervisor type

    // ... The types definitions are skipped ...

    // --- Allocated threads ---

    /// All classes of faults.
    pub HARD_FAULT;
    /// System service call.
    fn SV_CALL; // <-- add a new external interrupt handler
}
```

And also you will need to update your `src/bin.rs` to attach an external handler
for SVCall:

```rust
use drone_cortex_m::sv::sv_handler;
use CRATE_NAME::sv::Sv;

/// The vector table.
#[no_mangle]
pub static VTABLE: Vtable = Vtable::new(Handlers {
    reset,
    sv_call: sv_handler::<Sv>,
});
```

## Using processes

First, let's recall the generator fiber example from the previous chapter:

```rust
use core::pin::Pin;
use drone_cortex_m::{
    fib,
    fib::{Fiber, FiberState},
};

let mut fiber = fib::new(|| {
    let mut state = 0;
    while state < 3 {
        state += 1;
        yield state;
    }
    state
});
assert_eq!(Pin::new(&mut fiber).resume(()), FiberState::Yielded(1));
assert_eq!(Pin::new(&mut fiber).resume(()), FiberState::Yielded(2));
assert_eq!(Pin::new(&mut fiber).resume(()), FiberState::Yielded(3));
assert_eq!(Pin::new(&mut fiber).resume(()), FiberState::Complete(3));
```

This fiber can be rewritten using Drone process as follows:

```rust
use crate::sv::Sv;
use core::pin::Pin;
use drone_cortex_m::{
    fib,
    fib::{Fiber, FiberState},
};

let mut fiber = fib::new_proc::<Sv, _, _, _, _>(128, |_, yielder| {
    let mut state = 0;
    while state < 3 {
        state += 1;
        yielder.proc_yield(state);
    }
    state
});
assert_eq!(Pin::new(&mut fiber).resume(()), FiberState::Yielded(1));
assert_eq!(Pin::new(&mut fiber).resume(()), FiberState::Yielded(2));
assert_eq!(Pin::new(&mut fiber).resume(()), FiberState::Yielded(3));
assert_eq!(Pin::new(&mut fiber).resume(()), FiberState::Complete(3));
```

The difference is that the code inside the closure argument is fully
synchronous. The `proc_yield` call is translated to the `SVC` assembly
instruction. This instruction immediately switches the execution context back to
the caller. When the `resume` method of the process is called, it continues from
the last yield point, just like a generator.

The `fib::new_proc` function takes a stack size as the first argument. The stack
will be immediately allocated from the heap. To make this function safe, the
processor's MPU used to protect the stack from a possible overflow. On
processors without MPU, like STM32F103, this function will panic. However it is
still possible to use processes on such systems, though without any guarantees
about stack overflows. You can use `new_proc_unchecked` function, which is
marked `unsafe`.

Unlike generators, a process can take input data. And unlike `yield` keyword,
the `proc_yield` function not necessarily returns `()`. Here is an example of
such process:

```rust
let mut fiber = fib::new_proc::<Sv, _, _, _, _>(128, |input, yielder| {
    let mut state = input;
    while state < 4 {
        state += yielder.proc_yield(state);
    }
    state
});
assert_eq!(Pin::new(&mut fiber).resume(1), FiberState::Yielded(1));
assert_eq!(Pin::new(&mut fiber).resume(2), FiberState::Yielded(3));
assert_eq!(Pin::new(&mut fiber).resume(3), FiberState::Complete(6));
```
