# Introduction

Drone is an Embedded Operating System for writing real-time applications in
Rust. It aims to bring modern development approaches without compromising
performance into the world of embedded programming.

## Supported hardware

Drone core is platform-agnostic from the beginning. However currently only ARM®
Cortex®-M3/M4 is supported.

Drone also have special support for [Black Magic Probe](http://black-magic.org/)
in form of `drone bmp` commands. But it doesn't restrict you from using any
other debug probes.

## Design principles

- *Energy effective from the start*. Drone encourages interrupt-driven execution
  model.

- *Hard Real-Time*. Drone relies on atomic operations instead of using critical
  sections.

- *Fully preemptive multi-tasking with strict priorities*. A higher priority
   task takes precedence with minimal latency.

- *Highly concurrent.* Multi-tasking in Drone is very cheap, and Rust ensures it
  is also safe.

- *Message passing concurrency*. Drone ships with synchronization primitives out
  of the box.

- *Single stack by default*. Drone concurrency primitives are essentially
  stack-less state machines. But *stackful tasks are still supported*.

- *Dynamic memory enabled*. Drone lets you use convenient data structures like
  mutable strings or vectors while still staying deterministic and code
  efficient.

## Why use Drone?

- Async/await by default. Drone provides all required run-time to use native
  async/await syntax and execute `Future`s.

- Doesn't require `unsafe` code. In spite of the fact that Drone core inevitably
  relies on `unsafe` code, Drone applications can fully rely on the safe
  abstractions provided by Drone.

- Modern tooling. Apart from standard Rust tools like `cargo` package manager,
  `rustfmt` code formatter, `clippy` code linter, Drone provides `drone`
  command-line utility which can generate a new Drone project for your hardware,
  or manage your debug probe.

- Primary stack is stack-overflow protected regardless of MMU/MPU presence. But
  secondary stackful tasks require MMU/MPU to ensure the safety.

- Debug communication channels. Rust's `print!`, `eprint!` and similar macros
  are mapped to Cortex-M's SWO channels 0 and 1 out of the box. Debug messages
  incur no overhead when no debug probe is connected.

- `Drone.toml` configuration file, which saves you from manually writing linker
  scripts.

- Rich and safe zero-cost abstractions for memory-mapped registers. Drone
  automatically generates register bindings from vendor-provided SVD files. It
  also provides a way to write code generic over similar peripherals.

## What Drone doesn't

- Drone doesn't support loading dynamic applications. It is a library OS and is
  linked statically with its application.

- Drone doesn't implement time-slicing. It has a different execution model, but
  optional time-slicing may be added in the future.
