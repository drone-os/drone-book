# Message-Passing

The preferred way for inter-thread communication in Drone OS is
message-passing. In a similar way as Rust's stdlib offers `std::sync::mpsc` for
multi-producer single-consumer queues, Drone offers three different kinds of
single-producer single-consumer queues under `drone_core::sync::spsc`.

## Oneshot

The oneshot channel is used to transfer an ownership of a single value from one
thread to another. You can create a channel like this:

```rust
use drone_core::sync::spsc::oneshot;

let (tx, rx) = oneshot::channel();
```

`tx` and `rx` are transmitting and receiving parts respectively, they can be
passed to different threads. The `tx` part has a `send` method, which takes
`self` by value, meaning it can be called only once:

```rust
tx.send(my_message);
```

The `rx` part is a future, which means it can be `.await`ed:

```rust
let my_message = rx.await;
```

## Ring

For passing multiple values of one type, there is the ring channel. It works by
allocating a fixed-size ring-buffer:

```rust
use drone_core::sync::spsc::ring;

let (tx, rx) = ring::channel(100);
```

Here `100` is the size of the underlying ring buffer. The `tx` part is used to
send values over the channel:

```rust
tx.send(value1);
tx.send(value2);
tx.send(value3);
```

The `rx` part is a stream:

```rust
while let Some(value) = rx.next().await {
    // new value received
}
```

## Pulse

When you need to repeatedly notify the other thread about some event, but
without any payload, the ring channel might be an overkill. There is the pulse
channel, which is backed by an atomic counter:

```rust
use drone_core::sync::spsc::pulse;

let (tx, rx) = pulse::channel();
```

The `tx` part has a `send` method, which takes a number to add to the underlying
counter:

```rust
tx.send(1);
tx.send(3);
tx.send(100);
```

The `rx` part is a stream. Each successful poll of the stream clears the
underlying counter and returns the number, which was stored:

```rust
while let Some(pulses) = rx.next().await {
    // `pulses` number of events was happened since the last poll
}
```

## Futures and streams

Thread tokens have methods that helps creating described channels for connecting
with a particular thread.

`add_future` takes a fiber and returns a future (`rx` part of a oneshot
channel). The future will be resolved when the fiber returns `fib::Complete`:

```rust
use drone_cortexm::{fib, thr::prelude::*};

let pll_ready = thr.rcc.add_future(fib::new_fn(|| {
    if pll_ready_flag.read_bit() {
        fib::Complete(())
    } else {
        fib::Yielded(())
    }
}));
pll_ready.await;
```

`add_try_stream` returns a stream (`rx` part of a ring channel), which resolves
each time the fiber returns `fib::Yielded(Some(...))` or
`fib::Complete(Some(...))`:

```rust
use drone_cortexm::{fib, thr::prelude::*};

let uart_bytes = thr.uart.add_try_stream(
    100, // The ring buffer size
    || panic!("Ring buffer overflow"),
    fib::new_fn(|| {
        if let Some(byte) = read_uart_byte() {
            fib::Yielded(Some(byte))
        } else {
            fib::Yielded(None)
        }
    }),
);
```

`add_pulse_try_stream` returns a stream (`rx` part of pulse channel), which
resolves each time the fiber returns `fib::Yielded(Some(number))` or
`fib::Complete(Some(number))`:

```rust
use drone_cortexm::{fib, thr::prelude::*};

let sys_tick_stream = thr.sys_tick.add_pulse_try_stream(
    || panic!("Counter overflow"),
    fib::new_fn(|| fib::Yielded(Some(1))),
);
```
