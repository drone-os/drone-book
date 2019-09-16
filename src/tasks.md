# Tasks

In Drone OS applications, a task is a logical unit of work. Most often it's
represented as an `async` function that's running in a separate thread. By
convention, each task is placed into a separate module inside `src/tasks`
directory. The module contains at least a task main function named
`handler`. The function then re-exported in `src/tasks/mod.rs` like this:

```rust
pub mod my_task;

pub use self::my_task::handler as my_task;
```

It is common to use an unused interrupt as the task thread. For example, in
STM32F103, there is "UART5 global interrupt" at the position 53. If UART5
peripheral is not used by the application, its interrupt can be reused for a
completely different task:

```rust
thr::vtable! {
    // ... The header is skipped ...

    // --- Allocated threads ---

    /// All classes of faults.
    pub HARD_FAULT;
    /// A thread for `my_task`.
    pub 53: MY_TASK;
}
```

Then, assuming `my_task` is an `async` function, the thread can run the task as
follows:

```rust
use crate::tasks;
use drone_cortex_m::thr::prelude::*;

thr.my_task.enable_int();
thr.my_task.set_priority(0xB0);
thr.my_task.exec(tasks::my_task());
```

Now, whenever `my_task` future or any of its nested futures returns
`Poll::Pending`, the thread suspends. And it will be resumed when the future
will be ready for polling again. It is implemented by passing a
`core::task::Waker` behind the scenes, which will trigger the thread when
`wake`d.
