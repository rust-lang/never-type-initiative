# To `!`

This discusses the semantics for coercion into `!` from some expression.
Currently, these rules largely come down to coercion to `!` being possible
"after" an expression with `!` type is encountered, such as `return` or `break`.

FIXME: It seems like this may have subtle differences as the divergence
checking is done on HIR(?) but MIR may end up lowered into a different order --
should write out the rules and check more thoroughly. Relevant code:

* [`compiler/rustc_typeck/src/check/fn_ctxt/checks.rs:720`](https://github.com/rust-lang/rust/blob/2b681ac06b1a6b7ea39525e59363ffee0d1a68e5/compiler/rustc_typeck/src/check/fn_ctxt/checks.rs#L720)
  If we have *definitely* diverged, then a missing tail expression does not
  force the return type to be `()` (i.e., the implicit expression is not
  present).

  ```rust
  fn foo() -> u32 {
      return 0;
      // would normally be an error as function returns u32, not (), but accepted.
  }
  ```

FIXME: the above example does not feel obviously related to `!`, there's
probably some more reading and digging to do here.


Examples pulled from `src/test/ui/coercion/coerce-to-bang.rs` and `src/test/ui/coercion/coerce-to-bang-cast.rs`.

```rust
fn foo(x: usize, y: !, z: usize) { }

fn call_foo_a() {
    foo(return, 22, 44);
    //~^ ERROR mismatched types
}

fn call_foo_b() {
    // Divergence happens in the argument itself, definitely ok.
    foo(22, return, 44);
}

fn call_foo_c() {
    // This test fails because the divergence happens **after** the
    // coercion to `!`:
    foo(22, 44, return); //~ ERROR mismatched types
}

fn call_foo_d() {
    // This test passes because `a` has type `!`:
    let a: ! = return;
    let b = 22;
    let c = 44;
    foo(a, b, c); // ... and hence a reference to `a` is expected to diverge.
    //~^ ERROR mismatched types
}

fn call_foo_e() {
    // This test probably could pass but we don't *know* that `a`
    // has type `!` so we don't let it work.
    let a = return;
    let b = 22;
    let c = 44;
    foo(a, b, c); //~ ERROR mismatched types
}

fn call_foo_f() {
    // This fn fails because `a` has type `usize`, and hence a
    // reference to is it **not** considered to diverge.
    let a: usize = return;
    let b = 22;
    let c = 44;
    foo(a, b, c); //~ ERROR mismatched types
}

fn array_a() {
    // Return is coerced to `!` just fine, but `22` cannot be.
    let x: [!; 2] = [return, 22]; //~ ERROR mismatched types
}

fn array_b() {
    // Error: divergence has not yet occurred.
    let x: [!; 2] = [22, return]; //~ ERROR mismatched types
}

fn tuple_a() {
    // No divergence at all.
    let x: (usize, !, usize) = (22, 44, 66); //~ ERROR mismatched types
}

fn tuple_b() {
    // Divergence happens before coercion: OK
    let x: (usize, !, usize) = (return, 44, 66);
    //~^ ERROR mismatched types
}

fn tuple_c() {
    // Divergence happens before coercion: OK
    let x: (usize, !, usize) = (22, return, 66);
}

fn tuple_d() {
    // Error: divergence happens too late
    let x: (usize, !, usize) = (22, 44, return); //~ ERROR mismatched types
}

fn cast_a() {
    let y = {return; 22} as !;
    //~^ ERROR non-primitive cast
}

fn cast_b() {
    let y = 22 as !; //~ ERROR non-primitive cast
}
```

See some background/discussion in [rust-lang/rust#40800](https://github.com/rust-lang/rust/issues/40800).

# From `!`

Any coercion site with `!` as the "from" type always succeeds, with the
destination type being the new type. This includes inference variables.
