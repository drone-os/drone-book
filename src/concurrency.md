# Concurrency

Concurrency model is one of the most important aspects of an Embedded Operating
System. Applications for embedded micro-controllers require operating with
multiple sources of events at one time. Furthermore an embedded system should be
in a power-saving mode as often and as long as possible. Drone's goal is to make
writing highly concurrent and power-efficient applications easy and correct.

First, let's see how conventional Embedded Operating Systems work. They allow
you to create tasks that are running in parallel, each with its own stack:

![Conventional RTOS](../assets/conventional-rtos.svg)

However this is not how hardware is actually designed. In fact, processors can
only execute a single task at a time. What conventional Operating Systems
actually do, is that they are rapidly switching between tasks, to make them
**appear** to be running in parallel:

![Conventional RTOS Time Sharing](../assets/conventional-rtos-slices.svg)

That concurrency model, while having clear advantages for desktop and server
operating systems, incurs noticeable overhead for embedded real-time
systems. Also to protect from stack overflow errors it should be running on a
processor with built-in Memory Management/Protection Unit, which is not the case
for STM32F103.

Contrarily, modern hardware evolves in the direction of more elaborate interrupt
controllers. For example, Nested Vectored Interrupt Controller, or NVIC, which
can be found in each Cortex-M processor. It implements many hardware
optimizations to reduce scheduling costs, such as late-arriving or
tail-chaining. Drone OS utilizes such interrupt controllers to build strictly
prioritized fully preemptive scheduling:

![Drone Concurrency](../assets/drone-single-stack.svg)

Only a task with a higher priority can preempt another task. And a task must
completely relinquish the stack before completing or pausing to wait for an
event or a resource. This allows Drone OS to use a single stack for all program
tasks. This single stack is also protected from stack overflow errors by placing
it at the border of the RAM.

So how Drone achieves such stack usage for tasks? Mainly by using Rust's
async/await or generators syntax, which translate to state machines. The task
state, which needs to be saved between resumption points, is stored much more
compactly on the heap.

As an option Drone also implements conventional stateful tasks. Using such tasks
one can integrate an existing blocking code with a Drone application, by
allocating a separate stack. To use this feature safely, the processor must have
an MMU/MPU. Otherwise creating such task is `unsafe`, because the safety from
stack overflow couldn't be guaranteed.
