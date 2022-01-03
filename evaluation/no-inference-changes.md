This discusses the impact of just stabilizing `!` syntax and changing
`std::convert::Infallible` to be an alias for `!`. (That is, no inference
algorithm changes).

# Regression: inference failures

This leads to the regression identified in
[rust-lang/rust#66757](https://github.com/rust-lang/rust/issues/66757).

```rust
struct E;

impl From<!> for E {
    fn from(_: !) -> E {
        E
    }
}

#[allow(unreachable_code)]
fn foo(never: !) {
    <E as From<!>>::from(never);  // Ok
    <E as From<_>>::from(never);  // Inference fails here
}
```

Without changes to type inference fallback, code such as the above with
*explicit* `!` type now fails to compile. This happens because with `!`, unlike
with `std::convert::Infallible`, `!` can coerce to any variable. That means that
the coercion from `!` to `_` *succeeds*, leading to the inference failure here,
as there's no constraint placed on the argument.

The coercion to inference variables is *in general* desirable, because of code
like this:

```rust
if foo {
    loop {}
}
```

`if` expressions without else currently desugar such that the "block" is typed
at an inference variable (later eq'd with `()`), so the coercion to an inference
variable is necessary to make this work with the current compiler
implementation.  Further, changing the compiler to avoid this seems relatively
hard (FIXME: just how hard?). This regression is prominent enough that it makes
fully avoiding inference changes not possible.

# Con: misleading/confusing fallback

This leads to the painpoint (amongst others) identified in
[rust-lang/rust#66738](https://github.com/rust-lang/rust/issues/66738). This may
also be a regression in some cases if users start more explicitly writing `!` in
some places, though is not directly one at just stabilization time.

```rust
fn magic<R, F: FnOnce() -> R>(f: F) -> F { f }

fn main() {
    let f2 = magic(|| loop {}) as fn() -> !;
}
```

The (presumed) expected behavior is that `loop {}`, having type `!`, means that
the closure above will have a return type of `!`. However, that's not what
actually happens: the closure's return type is set to an inference variable,
`?T`. Since expressions in the return position are coerced to the return type,
we end up with no requirement on the return type's inference variable. (The cast
does not participate in inference, generally). This means that the inference
variable is unconstrained and so falls back to `()`.

Note that this behavior is currently relied upon by some libraries. For example,
code like this is relatively common, which requires that closures return `()` or
`u32`. This means that if the fallback doesn't go to `()`, we will end up with
an error.

```rust
trait Bar { }
impl Bar for () {  }
impl Bar for u32 {  }

fn foo<R: Bar>(_: impl Fn() -> R) {}

fn main() {
    foo(|| panic!());
}
```

Take a snippet from [warp 0.1.11](https://github.com/seanmonstar/warp/blob/v0.1.11/src/test.rs#L491-L496):

```rust
wr_rx.map_err(|()| {
    unreachable!("mpsc::Receiver doesn't error");
})
.forward(tx.sink_map_err(|_| ()))
.map(|_| ());
```

This code fails if the first closure is typed as `impl Fn(()) -> !`, because
`forward` requires that `!: From<()>` (i.e., that the error types are
compatible). That impl is not currently available, so this code doesn't work
out.

This kind of interaction is relatively common, which makes changes to closure
return types that "make sense" at an intuitive level actually commonly result in
errors.
