# PRs/Issues

This is intended to give a relatively quick link list to relevant issues/PRs or
individual comments, for possible referencing/inclusion elsewhere.

# Necessity of fallback changes

Canonical example of inference failure with no-inference-change scenario.

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

Discussion in:

* [#67225](https://github.com/rust-lang/rust/issues/67225)
* [#66757](https://github.com/rust-lang/rust/issues/66757)
    * [Possible options](https://github.com/rust-lang/rust/issues/66757#issuecomment-559771169)
    * [Necessity of fallback](https://github.com/rust-lang/rust/issues/66757#issuecomment-560571316)

# Conditional fallback

See [v1](../explainer/conditional-fallback-v1.md) explainer for the details of
the core algorithm.

Most of this code is implemented, gated on `never_type_fallback`,  landed in
[#88149](https://github.com/rust-lang/rust/pull/88149) (primarily refactoring) and
[#88804](https://github.com/rust-lang/rust/pull/88804) (most of the changes).

# Pain point / confusing behavior

* [Closure return types are `()` even if body `!`](https://github.com/rust-lang/rust/issues/66738)


