# Never type fallback proposal

This is a proposal for an alternative scheme for never type fallback. This scheme, while not fully backwards compatible, sidesteps the problems we've encountered so far in attempting to stabilize the `!` type:

* Unsound type inference changes from changing fallback, as described in [#66173]. These problems result from changing the fallback from `()` to `!` for type variables wind up interacting with "live code", resulting in `!` values being created.
* Regressions from having no fallback at all, as described in [#67225] and [#66757].

[#66173]: https://github.com/rust-lang/rust/issues/66173
[#67225]: https://github.com/rust-lang/rust/issues/67225
[#66757]: https://github.com/rust-lang/rust/issues/66757

## Background

The current fallback scheme is based on the concept of "diverging" type variables. In short, a `!` type can be coerced to any other type. But when it is inferred to an unknown type `?T` (a type variable), the way that we handle it is to create a *diverging* type variable `?D` and unify the two. Once type-checking is complete, we walk over any unbound type variables. If `?D` has not yet been unified with any concrete type (it may have been unified or related to other *general type variables*, but none of those type variables have yet been assigned a type), then it will "fallback" to a specified type. In current Rust, that type is `()`. The `!` type RFC proposed changing that fallback type to `!` (as part of introducing the concept of `!` as a standalone type). The idea behind this fallback is that since `?D` represents the type of an expression that is known to diverge, what actual type it is assigned to doesn't matter, since it can never affect live code. Unfortunately, this premise is false.

### Bug 66173: Unsoundness introduced by changing fallback from `!` to `()`

As described in [#66173], we have found in practice that the fallback of diverging type variables *can* impact the types assigned to live code. The most common problem involves match patterns. Consider the following pattern:

```rust
let x = match something() {
    pattern1 => Default::default(),
    pattern2 => panic!("..."),
};
```

In a case like this, the type of the match is ultimately represented by a type variable `?M`. The first arm is assigned a type variable `?T` and the second arm (which panics) gets the type `!`. Both `?T` and `!` are coerced into `?M`:

```
?T -> ?M
! -> ?M
```

The first coercion creates a subtyping relationship (`?T <: ?M`) because the two types are unknown. The second coercion creates a diverging type variable `?D` and a subtyping relationship `?D <: ?M`.

The problem now is that if `?D` falls back to `!`, then this winds up causing `?M` and `?T` to both be assigned the type `!`. In this *particular* example the result is a compilation error, because `Default` is not implemented for `!`, but in [other examples](https://github.com/rust-lang/rust/issues/66173#issuecomment-574892360) the result was unsound execution.

This example prompted us to hold off on changing the fallback from `()` to `!`. The result is in some way no less surprising: the type of `Default::default` winds up falling back to `()`, rather than (say) requiring an explicit type annotation. However, at least using `()` didn't produce unsound behavior in previously sound code.

### Bug 66757: Regressions introduced by NOT changing fallback

Unfortunately, if we don't change the fallback to `!`, then we also trigger other sorts of regressions (at least if we want to also redefine the `Infallible` type in the stdlib). As described in [#66757], the fundamental problem is that we *have* a `From<!>` impl for any type `E`, but we *don't* have a `From<()>` impl. So when we have code that requires `From<?D>` where `?D` is a diverging type variable, falling back to `!` is preferred.

This is related to the fact that `!` is in many cases the *right* fallback! If you have code like `Some(return)`, you would prefer that the type of this expression (if not otherwise constrained) is `Option<!>`, not `Option<()>`. In the case of [#66757], we had similar code like `From::from(never)` where `never: Infallible`. If `Infallible` is an alias for `!`, this ought to "fallback" to `!`.

## Proposal: fallback chooses between `()` and `!` based on the coercion graph

So we've seen that changing the fallback *from* `()` causes unsoundness, but *keeping* the fallback as `!` can cause failed compilations. The proposal in this PR is to cause the fallback to be more subtle: diverging type variables prefer to fallback to `!` but *sometimes* fallback to `()` (in cases where they may leak out into live code, in particular).

The idea is based on a "coercion graph". Roughly speaking, each type that an unbound type variable `?A` is coerced into another unbound type variable `?B`, we create a `Coercion(?A -> ?B)` relation (instead of immediately falling back to subtyping). At the end of type-checking, we can take any such relations that remain (because neither `?A` nor `?B` was constrained to another type) and create a graph where an edge `?A -> B` indicates that `?A` is coerced into `?B`.
Similarly, we can identify those type variables `?X` where we have a coercion `! -> ?X`. We call those *diverging* type variables. Each *diverging* type variable will either fallback to `!` or `()` depending on the coercion graph:

* Let `D*` be the set of type variables that are reachable from a diverging type variable via edges in the coercion graph. These are therefore the variables where the `!` type "flows into" them (or would, if it didn't represent the result of a diverging execution). 
* Let `N` be the set of type variables that are (a) unresolved and (b) not a member of `D*`. 
* Let `N*` be the set of type variables that are reachable from `N`. 
* Each diverging type variable in `D` will fallback to `()` if it can reach a variable in `N*` in the coercion graph, and otherwise fallback to `!`. 

The intuition here is: if there is some group of unconstrained type variables `?X` that are all dominated in the coercion graph by the type `!`, then they fallback to `!`. If there are type variables in the coercion graph that are the *target* of a `!` coercion but *also* flow into variable that are the target of other coercions, they fallback to `()`. 

### Effect on [#66173]

Recall the example from [#66173]:

```rust
let x: /*?X*/ = match something() {
    pattern1 => Default::default(), // has type `?T`
    pattern2 => panic!("..."), // has type `!`
}; // the match has type `?M`
```

Here we have a coercion graph as follows:

```
?T -> ?M
! -> ?M
?M -> ?X
```

In this case, applying the rules above:

* The set `D` of diverging variables is `[?M]`
* The set `D*` of variables reachable from `D` is `[?M, ?X]`
* The set `N` of non-diverging variables is `[?T]`
* The set `N*` of variables reachable from `N` is `[?X, ?X, ?T]`
* Since the diverging variable `?M` can reach a variable in `N*`, it falls back to `()`, and the unsoundness is averted.

### Effect on [#66757]

Recall this example much like [#66757]:

```rust
<?R as From<?F>>::from(return)
```

Here we have a coercion graph as follows:

```
! -> ?F
```

In particular, the type of the argument is `?F` and it is the target of a coercion from `!` (the type of the `return` expression). Since `?F` is *only* reachable from a `!`, it falls back to `!` as desired.

### Weird cases

There are some "weird cases" where `()` fallback can result even in dead code.

```rust
let mut x;
x = return;
x = Default::default();
```

Here, the type `?X` of `x` will have an "incoming edge" from the result of `Default::default` which will cause it to fallback to `()`. Seems ok.

### Backwards incompatibilies

Changing the fallback from *always* preferring `()` to *sometimes* preferring `!` can still cause regressions:

```rust
trait Foo { }
impl Foo for () { }
impl Foo for i32 { }
fn gimme<F: Foo>(f: F) { }
fn main() {
    gimme(return);
}
```

Here, the type argument of `gimme(return)` will fallback to `!` and stop compiling. 

It can also cause changes in behavior, though that is relatively difficult to engineer. An example might be:

```
match true {
    true => Cell::new(Default::default()),
    false => Cell::new(return),
}
```

In this case, the type variable for `Default::default` is never directly coerced into the type variable for `return`, so the latter would still fallback to `!`. In this example that would cause a compilation failure but one could imagine variants, similar to #66173, which would be unsound. (It's worth noting that the lint which @blitzerr and I were experimenting with *would* detect cases like this, though unfortunately it also detected its fair share of false warnings.) 

## Future extensions

I would like to deprecate the `()` fallback. I believe that these cases ought to be hand-annotated and are quite surprising. Consider Example 1 from [this github comment](https://github.com/rust-lang/rust/issues/66173#issuecomment-720527102):

```rust
parser.unexpected()?;
```

Here the `?` desugaring winds up causing this to fall back to `()`, but I think it should require manual annotation, as removing the `?` would require manual annotation. We can consider this separately but this branch should make it possible to do such a transition over an edition, perhaps.
