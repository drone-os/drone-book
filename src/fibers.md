# Fibers

Fibers in Drone OS are essentially finite-state machines. On type level, a fiber
is an instance of an anonymous type, which implements the `Fiber` trait. The
trait is defined at `drone_core::fib` as follows:

```rust
pub trait Fiber {
    type Input;
    type Yield;
    type Return;

    fn resume(
        self: Pin<&mut Self>,
        input: Self::Input,
    ) -> FiberState<Self::Yield, Self::Return>;
}

pub enum FiberState<Y, R> {
    Yielded(Y),
    Complete(R),
}
```

`Fiber` and `FiberState` are similar to `Generator` and `GeneratorState` from
`core::ops`, but with addition of the input parameter. Also like generators, it
is invalid to resume a fiber after completion.

A fiber can be created in multiple ways using `drone_cortex_m::fib::new_*`
family of constructors. For example a fiber that completes immediately upon
resumption can be created from an `FnOnce` closure:

```rust
use core::pin::Pin;
use drone_cortex_m::{
    fib,
    fib::{Fiber, FiberState},
};

let mut fiber = fib::new_once(|| 4);
assert_eq!(Pin::new(&mut fiber).resume(()), FiberState::Complete(4));
```

A fiber that involves multiple yield points before completion can be created
from an `FnMut` closure:

```rust
let mut state = 0;
let mut fiber = fib::new_fn(move || {
    if state < 3 {
        state += 1;
        fib::Yielded(state)
    } else {
        fib::Complete(state)
    }
});
assert_eq!(Pin::new(&mut fiber).resume(()), FiberState::Yielded(1));
assert_eq!(Pin::new(&mut fiber).resume(()), FiberState::Yielded(2));
assert_eq!(Pin::new(&mut fiber).resume(()), FiberState::Yielded(3));
assert_eq!(Pin::new(&mut fiber).resume(()), FiberState::Complete(3));
```

Or an equivalent fiber can be created using Rust's generator syntax:

```rust
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

The fibers described in this chapter are the main building blocks for Drone OS
tasks. But there is one more type of fibers, which will be described in the next
chapter.
