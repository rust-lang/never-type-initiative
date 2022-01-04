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
* Missing `From<!> for T`
    * Conflicts with `From<T> for <T>`
    * After stabilizing `!`, a crate can add `From<!> for MyType` preventing a
      std impl.
* Uninhabited enums (e.g., for errors)

# Miscellaneous

* Proposed (and rejected) change of fallback to `!`
  [rust-lang/rust#40801](https://github.com/rust-lang/rust/issues/40801).
* Propagating coercions more 'deeply'
  [rust-lang/rust#40924](https://github.com/rust-lang/rust/issues/40924)
  May help avoid some of the current problems of over-eagerly coercing into
  inference variables, for example.
* Prohibit coercion to `!` from other types in trailing expressions after a
  diverging expression
  [rust-lang/rust#46325](https://github.com/rust-lang/rust/issues/46325)
  * Affected
    [rust-lang/rust#40224](https://github.com/rust-lang/rust/pull/40224),
    removing the type-check success case discussed therein.
* `resolve_trait_on_defaulted_unit` lint
  [rust-lang/rust#39216](https://github.com/rust-lang/rust/issues/39216)
  * [Regression issue](https://github.com/rust-lang/rust/issues/51125)
