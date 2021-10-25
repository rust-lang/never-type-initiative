# ✨ RFC

- Feature Name: never_type, never_type_fallback
- Start Date: TBD
- RFC PR: Past [rust-lang/rfcs#1216](https://github.com/rust-lang/rfcs/pull/1216)
- Rust Issue: TBD

# Summary

Promote `!` to be a full-fledged type equivalent to an `enum` with no variants,
and adjust default inference fallback to avoid breakage.

# Guide level explanation

While empty types exist already in Rust today, for example in the form of `enum
Foo {}`, most Rust users will not be directly familiar with them, so this RFC
provides an explanation of the properties these types hold (and that `!` will
hold).

  * **They never exist at runtime**, because there is no way to create one.

  * **Code that handles them cannot execute**, because there is no value that it
    could execute with. Therefore, having an empty type in scope is a static
    guarantee that a piece of code will never be run.

  * **They represent the return type of functions that don't return**. For a
    function that never returns, such as `std::process::exit`, the set of all
    values it may return is the empty set. That is to say, the type of all
    values it may return is the type of no inhabitants, ie. `Never` or anything
    isomorphic to it. Similarly, they are the logical type for expressions that
    never return to their caller such as `break`, `continue` and `return`.

  * **They can be converted to any other type**.
    To specify a function `A -> B` we need to specify a return value in `B` for
    every possible argument in `A`. For example, an expression that converts
    `bool -> T` needs to specify a return value for both possible arguments
    `true` and `false`:

    ```rust
    let foo: &'static str = match x {
      true  => "some_value",
      false => "some_other_value",
    };
    ```

    Likewise, an expression to convert `() -> T` needs to specify one value,
    the value corresponding to `()`:

    ```rust
    let foo: &'static str = match x {
      ()  => "some_value",
    };
    ```

    And following this pattern, to convert `Never -> T` we need to specify a
    `T` for every possible `Never`. Of which there are none:

    ```rust
    let foo: &'static str = match x { };
    ```

    Reading this, it may be tempting to ask the question "what is the value of
    `foo` then?". Remember that this depends on the value of `x`. As there are
    no possible values of `x` it's a meaningless question and besides, the
    fact that `x` has type `Never` gives us a static guarantee that the match
    block will never be executed.

Here's some example code that uses `Never`. This is legal rust code that you
can run today.

```rust
use std::process::exit;

// Our empty type
enum Never {}

// A diverging function with an ordinary return type
fn wrap_exit() -> Never {
    exit(0);
}

// we can use a `Never` value to diverge without using unsafe code or calling
// any diverging intrinsics
fn diverge_from_never(n: Never) -> ! {
    match n {
    }
}

fn main() {
    let x: Never = wrap_exit();
    // `x` is in scope, everything below here is dead code.

    let y: String = match x {
        // no match cases as `!` is uninhabited
    };

    // we can still use `y` though
    println!("Our string is: {}", y);

    // we can use `x` to diverge
    diverge_from_never(x)
}
```

This RFC proposes that we allow `!` to be used directly, as a type, rather than
using `Never` (or equivalent) in its place. Under this RFC, the above code
could more simply be written.

```rust
use std::process::exit;

fn main() {
    let x: ! = exit(0);
    // `x` is in scope, everything below here is dead code.

    let y: String = match x {
        // no match cases as `Never` has no variants
    };

    // we can still use `y` though
    println!("Our string is: {}", y);

    // we can use `x` to diverge
    x
}
```

# Motivation

There are several key motivators for adding `!`, despite the existence of empty
types in the language already.

## It creates a standard empty type for use throughout Rust code

Empty types are useful for more than just marking functions as diverging.
When used in an enum variant they prevent the variant from ever being
instantiated. One major use case for this is if a method needs to return a
`Result<T, E>` to satisfy a trait but we know that the method will always
succeed.

For example, here's a possible implementation of `FromStr` for `String` than
currently exists in `libstd`.

```rust
impl FromStr for String {
    type Err = !;

    fn from_str(s: &str) -> Result<String, !> {
        Ok(String::from(s))
    }
}
```

This result can then be safely unwrapped to a `String` without using
code-smelly things like `unreachable!()` which often mask bugs in code.

```rust
let r: Result<String, !> = FromStr::from_str("hello");
let s = match r {
    Ok(s)   => s,
    Err(e)  => match e {},
}
```

The `Try` trait family is written with never type in mind, and currently
relies on the `enum Infallible {}` defined in libcore as a substitute.

Empty types can also be used when someone needs a dummy type to implement a
trait. Because `!` can be converted to any other type it has a trivial
implementation of any trait whose only associated items are non-static
methods. The impl simply matches on self for every method.

Example:

```rust
trait ToSocketAddr {
    fn to_socket_addr(&self) -> IoResult<SocketAddr>;
    fn to_socket_addr_all(&self) -> IoResult<Vec<SocketAddr>>;
}

impl ToSocketAddr for ! {
    fn to_socket_addr(&self) -> IoResult<SocketAddr> {
        match self {}
    }

    fn to_socket_addr_all(&self) -> IoResult<Vec<SocketAddr>> {
        match self {}
    }
}
```

All possible implementations of this trait for `!` are equivalent. This is
because any two functions that take a `!` argument and return the same type
are equivalent - they return the same result for the same arguments and
have the same effects (because they are uncallable).

Suppose someone wants to call `fn foo<T: SomeTrait>(arg: Option<T>)` with
`None`. They need to choose a type for `T` so they can pass `None::<T>` as
the argument. However there may be no sensible default type to use for `T`
or, worse, they may not have any types at their disposal that implement
`SomeTrait`. As the user in this case is only using `None`, a sensible
choice for `T` would be a type such that `Option<T>` can ony be `None`, i.e.
it would be nice to use `!`.

While the trait author or user could define their own empty type and
implement the trait themselves, it is useful to avoid the hassle of needing
to import the appropriate type in order to specify it in cases like this.

## Better dead code detection

Consider the following code:

```rust
let t = std::thread::spawn(|| panic!("nope"));
t.join().unwrap();
println!("hello");
```

Under this RFC: the closure body gets typed `!` instead of `()`, the `unwrap()`
gets typed `!`, and the `println!` will raise a dead code warning. This
requires a canonical empty type to fallback to for the type of expressions
like `panic!()`.

To be clear, `!` has a meaning in any situation that any other type does. A `!`
function argument makes a function uncallable, a `Vec<!>` is a vector that can
never contain an element, a `!` enum variant makes the variant guaranteed never
to occur and so forth. It might seem pointless to use a `!` function argument or
a `Vec<!>`, but that's no reason to disallow it. And generic code sometimes
requires it.

It's also worth noting that the `!` proposed here is *not* the bottom type that
used to exist in Rust in the very early days. Making `!` a subtype of all types
would greatly complicate things as it would require, for example, `Vec<!>` be a
subtype of `Vec<T>`. This `!` is simply an empty type (albeit one that can be
cast to any other type).

# Reference-level explanation

Add a new primitive type `!` to Rust. `!` behaves like an empty enum except that
it can be implicitly cast to any other type. For example, code like the
following is acceptable:

```rust
let r: Result<i32, !> = Ok(23);
let i = match r {
    Ok(i)   => i,
    Err(e)  => e, // e is cast to i32
}
```

We also adjust the type inference algorithm used by rustc to fallback to `!`
rather than `()` when an inference variable ends up unconstrained. This change
is motivated by the desire to provide a better experience for Rust users since
expressions typed at `!` can be coerced into other types. If we avoid making
this change, we have to specify that expressions like `loop {}` and `panic!()`
continue to "return" `()`, which seems obviously wrong.

Adjusting type inference to fallback to `!` also helps set us on a path towards
removing some of the fallback and *requiring* explicit types in the future, as
we can be more confident that the code involved is somewhat faulty.

## Ensuring soundness

However, this change to fallback is not generally sound in the presence of
unsafe code that is present in the wild today. The following is a piece of
example code that was previously OK, but with the proposed change has undefined
behavior.  It's a stripped down example -- the code is already unsound in the
sense that `unconstrained_return` must only be called with valid `T`, but in
practice code like this exists and does not cause UB (as it never gets
instantiated with an incorrect `T`) with today's compiler.

```rust
fn unconstrained_return<T>() -> Result<T, String> {
    let ffi: fn() -> T = std::mem::transmute(some_pointer);
    Ok(ffi())
}

fn foo() {
    let _a = match unconstrained_return::<_>() {
        Ok(a) => a,
        Err(s) => panic!("failed to run"),
    };
}
```

The match here has type `?M` and the two arms have types `?T` and `!`, as
`panic!` has type `!` with these changes. `?T` here is effectively
unconstrained, so it falls back to `!` with the proposed change to type
inference. As a result, `unconstrained_return`'s call to `ffi()` has a return
type of `!`, which means that the compiler is free to assume that the function
does not return. Since `ffi()` in practice does return, this leads to UB.

So we cannot simply change fallback in all cases.

We propose an algorithm which indicates whether it is safe to fallback to `!`.

For each inference variable still present during fallback (i.e. those which are
unconstrained), we look at the relationships between these inference variables
to determine whether to keep the fallback to `()` or fallback to `!`.

We construct a directed graph where the vertices are inference variables, and
add edges `?A -> ?B` if `?A` is coerced into `?B`.

Let `D` be the set of type variables `?X` which were created as a result of the
coercion of `! -> ?X` being proposed. These can arise from functions returning
`!` or `loop {}` expressions, which can coerce into any type.

For each type variable `?T`, we fallback to:

* `!` if the variable is dominated in the graph by the set `D` (that is, there
  is no path from a non-diverging type variable to it).
* `()` otherwise.

The intuition here is that variables reachable only from diverging type
variables must be in dead code, but if they are *also* reachable from some
non-diverging type variable, they may be in live code, and so fallback to `!` is
unsound.

## Avoiding regressions

The proposed graph is sufficient for soundness, but unfortunately is
insufficient to avoid a good number of regressions. We add an additional
heuristic that mitigates most of the remaining breakage (~100 crates per
Crater), leaving ~25 regressions.

This heuristic adds another case of fallback of `?T` to `()` when we encounter the
following:

* `?T: Foo` and
* `Bar::Baz = ?T` and
* `(): Foo`

This pattern is commonly encountered in code which constrains the return type of
some closure, and where closure instances frequently contain `panic!()`. With
the previously proposed fallback, there is no additional constraint on `?T`
(it's not flowing into any other inference variables), so it is reasonable for
it to fallback to `!`. However, in practice, that causes breakage, as the
closure's return type is restricted to `Bar`, which is only implemented for `()`
(and `u32`).

```rust
// Crate A:
pub trait Bar { }
impl Bar for () {  }
impl Bar for u32 {  }

pub fn foo<R: Bar>(_: impl Fn() -> R) {}

// Crate B:
fn main() {
    foo(|| panic!());
}
```

# Alternatives

## Leave `Infallible` as the standard empty type

We could abandon the effort to stabilize `!` as a dedicated return type, instead
leaving the currently existing `Infallible` as the standard empty type. There
are several downsides to this:

* Users need to learn two different canonical names (`!` if standalone in a function return
  types and `Infallible` for other places)
* `Infallible` is either special-cased to coerce to other types or users need to
  write `match foo {}` to get that behavior.

## Edition-based fallback?

We considered whether the fallback algorithm could be adjusted over an edition
boundary to permit stabilizing never type. In practice, this doesn't work
because the never type is a vocabulary type and so is present in the public API
of various crates (including core/std, after stabilization).

This means that while the fallback within a library might be edition-dependent,
we'd still need to stabilize full use of `!` across all editions. That basically
puts prior editions on the same grounds as *just* stabilizing `!`, which is
known to introduce regressions.

Generally it seems pretty clear that stabilizing never type (or fallback
changes) only on a subset of editions would lead to a more challenging landscape
for Rust users. For example, as we've seen, fallback has implications for
soundness, so we'd still need all of the complexity in the fallback algorithm in
order to prevent cases like the one described above -- this seems to just make
things more complicated, rather than improving them.

# Future work

## Removing (parts of) fallback

We can consider removing some of the cases of fallback, particularly those which
cause problems for this RFC, in the future. There were several examples
encountered in Crater and elsewhere that required effort to deduce the inferred
type. Additionally, we believe that it's likely that cases of fallback to `()`
in new code are likely better written in other ways and we may wish to force the
user to indicate a type rather than silently providing one that may cause
trouble or hurt diagnostic quality.

We can leverage editions to remove parts of fallback. Unlike the phase-in of
never type fallback, *removing* fallback only prevents code from compiling and
does not change the behavior of existing code. This is fairly standard for
cross-edition changes, and so is much more straightforward to design.

However, this RFC does not propose a particular path towards removing/improving
our fallback algorithms, beyond noting that it may be advantageous to do so. If
we do so, we can likely end up avoiding the more complex fallback algorithm
proposed by this RFC being a permanent feature of future Rust editions.

## Special casing trait impls for !

`!` is useful as a standard marker inside types which may derive various traits.
Implementing a trait for `!` is frequently fairly annoying, as you need to write
out many functions which are largely inconsequential as they can never run (if
they have a `self` parameter).

For now, the expectation is that library authors will want to add manual
impls for standard traits (including in the standard library), but a future
extension could let users write (potentially) overlapping trait impls for `!` so
long as the body of the trait is "obviously equivalent" as the methods can't be
called.

# Unresolved questions

* Pending (needs review of the above)
