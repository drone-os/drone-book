# Memory-Mapped Registers API Summary

This section provides examples of most common methods on register and field
tokens. For complete API refer to `drone_core::reg` and `drone_cortexm::reg`
module docs.

## Whole Registers

Read the value of RCC_CR register:

```rust
let val = rcc_cr.load();
```

HSIRDY is a single-bit field, so this method returns a `bool` value indicating
whether the corresponding bit is set or cleared:

```rust
val.hsirdy() // equivalent to `val & (1 << 1) != 0`
```

HSITRIM is a 5-bit field in the middle of the RCC_CR register. This method
returns an integer of only this field bits shifted to the beginning:

```rust
val.hsitrim() // equivalent to `(val >> 3) & ((1 << 5) - 1)`
```

Reset the register RCC_CR to its default value, which is specified in the
reference manual:

```rust
rcc_cr.reset();
```

The following line writes a new value to the RCC_CR register. The value is the
register default value, except HSION is set to 1 and HSITRIM is set to 14.

```rust
rcc_cr.store(|r| r.set_hsion().write_hsitrim(14));
// Besides "set_", there are "clear_" and "toggle_" prefixes
// for single-bit fields.
```

And finally the following line is a combination of all of the above, it performs
read-modify-write operation:

```rust
rcc_cr.modify(|r| r.set_hsion().write_hsitrim(14));
```

Unlike `store`, which resets unspecified fields to the default, the `modify`
method keeps other field values intact.

## Register Fields

If you have only a register field token, you can perform operations affecting
only this field, and not the other sibling fields:

```rust
rcc_cr_hsirdy.read_bit(); // equivalent to `rcc_cr.load().hsirdy()`
rcc_cr_hsitrim.read_bits(); // equivalent to `rcc_cr.load().hsitrim()`
rcc_cr_hsirdy.set_bit(); // equivalent to `rcc_cr.modify(|r| r.set_hsirdy())`
rcc_cr_hsirdy.clear_bit(); // equivalent to `rcc_cr.modify(|r| r.clear_hsirdy())`
rcc_cr_hsirdy.toggle_bit(); // equivalent to `rcc_cr.modify(|r| r.toggle_hsirdy())`
rcc_cr_hsitrim.write_bits(14); // equivalent to `rcc_cr.modify(|r| r.write_hsitrim(14))`
```

Also if you have tokens for several fields of the same register, you can perform
a single read-modify-write operation:

```rust
rcc_cr_hsion.modify(|r| {
    rcc_cr_hsion.set(r);
    rcc_cr_hsitrim.write(r, 14);
});
// Which would be equivalent to:
rcc_cr.modify(|r| r.set_hsion().write_hsitrim(14));
```
