# Testing

Testing Embedded Systems is more difficult than testing standard
applications. There are at least two hardware platforms involved: the one that
runs the compiler, and the target system. Testing on the development machine is
much easier, but it can't test hardware-specific code. Conversely, testing
directly on the target system is much harder and requires elaborate hardware
setup.

Drone OS supports testing on the development machine out of the box. Drone
crates as well as all projects generated with `drone new` have a special
feature, named `std`. When you run the test recipe:

```shell
$ just test
```

Your program is compiled for your development machine target (usually
`x86_64-unknown-linux-gnu`), and not for your device target
(e.g. `thumbv7m-none-eabi`). And the program is compiled with the `std` feature
enabled. This allows to run standard Rust's test runner.

This way you can use all standard Rust testing options: inline `#[test]`
functions, separate test files under `tests/` directory, documentation tests
(including `compile_fail` tests.) Also your tests have access to the `std`
crate.

Though, you should keep in mind that the pointer size in your tests and in the
release code will usually differ. This kind of tests is suitable for testing
algorithms and business logic. Hardware-specific code often will not even
compile. For this, you should use condition compilation like in this snippet
from `drone-cortexm`:

```rust
fn wait_for_int() {
    #[cfg(feature = "std")]
    return unimplemented!();
    unsafe { asm!("wfi" :::: "volatile") };
}
```
