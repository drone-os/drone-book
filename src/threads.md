# Threads

A thread in Drone OS corresponds to a hardware interrupt. It is a sequence of
fibers that managed independently by an interrupt controller. Threads can not be
created on demand, but should be pre-defined for a particular project. Then any
number of fibers can be attached dynamically to a particular thread.

Threads should be defined at `src/thr.rs` using `thr!` and `thr::vtable!`
macros:

```rust
thr::vtable! {
    use Thr;

    /// The vector table type.
    pub struct Vtable;

    /// Explicit vector table handlers.
    pub struct Handlers;

    /// A set of thread tokens.
    pub struct Thrs;

    /// The array of thread data.
    static THREADS;

    // --- Allocated threads ---

    /// All classes of faults.
    pub HARD_FAULT;
    /// A thread for my task.
    pub 10: MY_THREAD;
}

thr! {
    use THREADS;

    /// The thread data.
    pub struct Thr {}

    /// The thread-local storage.
    pub struct ThrLocal {}
}
```

The macros will define `THREADS` static array of `Thr` objects. In this example
the array will contain three element: `HARD_FAULT`, `MY_THREAD`, and the
implicit `RESET` thread data. `Thrs` structure is also created here, which is a
zero-sized type, a set of tokens, through which one can manipulate the
threads. This set of token can be instantiated only once, usually at the very
beginning of the root task:

```rust
/// The root task handler.
#[inline(never)]
pub fn handler(reg: Regs, thr_init: ThrsInit) {
    let thr = thr::init(thr_init);

    // ... The rest of the handler ...
}
```

Here the `thr` variable contains tokens for all defined threads. If you have
added fields to the `Thr` definition, they are accessible through
`thr.my_thread.to_thr()`. `ThrLocal` is also stored inside `Thr`, but accessible
only through the `thr::local()` free-standing function.

A thread can be called programmatically using implicit `core::task::Waker` or
explicit `thr.my_thread.trigger()` or directly by hardware peripherals. If the
thread, which was triggered, has a higher priority than the currently active
thread, the active thread will be preempted. If the thread has a lower priority,
it will run after all higher priority threads. Priorities can be changed on the
fly with `thr.my_thread.set_priority(...)` method.

## Fiber chain

The main thing a thread owns is a fiber chain. A fiber chain is essentially a
linked list of fibers. A fiber can be added to a thread chain dynamically using
`thr.my_thread.add_fib(...)`, or other methods based on it. The `add_fib` method
is atomic, i.e. fibers can be added to a particular thread from other threads.

When a thread is triggered, it runs the fibers in its fiber chain one-by-one in
LIFO order. In other words the most recently added fiber will be executed
first. A fiber can return `fib::Yielded` result, which means the fiber is paused
but not completed; the thread will keep the fiber in place for the later run and
proceed with the next fiber in the chain. Or the fiber can return
`fib::Complete`, in which case the thread removes the fiber from the chain, runs
its `drop` destructor, and proceeds to the next fiber in the chain.
