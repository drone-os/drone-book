# Platform-Specific Layer

This layer fills the gap between the platform-agnostic core and the specific
platform. Target-specific intrinsics and inline assembler can be used here.

The platform crate implements `drone-core` runtime at `src/rt.rs`. Furthermore
it can export various utility functions. For example
`drone_cortexm::processor::self_reset`, which runs a sequence of assembly
instructions to issue a self-reset request.

This layer provides a backend for Drone's threading API. If we take Cortex-M as
an example, here the Drone threading system is implemented by leveraging NVIC
(Nested Vectored Interrupt Controller.) Where each Drone thread corresponds to a
hardware interrupt, and NVIC is responsible for switching between the threads.

The crate should include at least one `core::task::Waker` implementation for
Rust `Future`s. `drone-cortexm` implements two: one for the lowest thread, which
utilizes `WFE`/`SEV` assembly instructions, and the other uses the `NVIC_STIR`
register.

As stackful threading is highly target-specific, stackful Drone fibers are
implemented at this layer. If the target incorporates an MPU (Memory Protection
Unit), it should be used to protect from stack overflow errors. Because the core
Drone provides zero-cost protection only for the main stack, and hence only for
stackless fibers. If there is no MPU, the corresponding constructor functions
must be marked `unsafe`.
