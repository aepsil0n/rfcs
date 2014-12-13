- Start Date: 2014-04-29
- RFC PR #:
- Rust Issue #:


# Summary

Implement a simple form of a dependent type system similar to C++ and D by
allowing generics to have static values as parameters in addition to lifetime
and type parameters.


# Motivation

Generic types are very useful to ensure type-safety while writing generic code
for a variety of types. Parametrisation over static values extends this feature
to certain use cases, where it is necessary to maintain type-safety in terms of
these values.

To illustrate this further, consider the following use cases for this feature:

* *Algebraic types*: algebraic vectors and matrices generally have a certain
  dimensionality, which changes their behaviour. For example, it does not make
  sense to add a 2-vector with a 3-vector or multiply a 3x4 matrix by a
  5-vector. But the algorithms can be written generically in terms of the
  dimensionality of these types.
* *Compile-time sizes*: define sizes for small buffers and other data
  structures at compile-time. This can be quite important to only allocate
  larger chunks of memory, when they are really required. If this is unlikely,
  it can have a meaningful impact on performance. For instance,
  [Servo][servo_macros] currently resorts to a range of types implemented with
  macros to deal with this issue.
* *Physical units*: In science one often deals with numerical quantities
  equipped with units (meters, hours, kilometers per hour, etc.). To avoid
  errors dealing with these units, it makes sense to include in the data type.
  For performance reasons, however, having the computational overhead in every
  single calculation at run-time might be prohibitively slow. In this context
  value parameters could allow to convert between units and check formulae for
  consistency at compile-time.
* *Range and mod types*: This would allow [range and mod types similar to
  Ada][range_mod_ada] in Rust, which enforce certain range constraints and
  implement modular arithmetics respectively.

In general, this feature allows for nicer abstractions that are dealt with at
compile-time and thus have no cost in terms of run-time performance. It is also
a very expressive feature that Rust is currently lacking in comparison with
similar languages such as C++ and D.


# Drawbacks

First of all, allowing another kind of parameter for generics adds a certain
degree of complexity to Rust. Thus the potential merits of this feature have to
justify this.

Furthermore, it is not entirely clear, how well this feature fits with Rust's
current approach to meta-programming. When implemented completely, this feature
requires compile-time function execution (CTFE), which has been
[discussed][issue_11621] [in the past][ctfe_mail] without a clear outcome. This
feature would also introduce the possibility for
[template metaprogramming][template_meta]. However, Rust's type system is
already [Turing-complete][turing_complete].


# Detailed design

## Basic syntax

Currently generics can be parametrized over type and lifetime parameters. This
RFC proposes to additionally allow for static values as parameters. These
values must be static, since they encode type-information that must be known at
compile-time.

To propose a concrete syntax, consider these examples of a function, a struct
and an enum:

```rust
// Add a static integer to another one
fn add_n<static n: int>(x: int) -> int {
    x + n
}

// An n×m matrix
struct Matrix<T, static n: uint, static m: uint> {
    arr: [[T, ..n], ..m],
}

```

The syntax `"static" ident ':' type` closely resembles the syntax for type parameters
with trait bounds. There is the additional keyword `static` to clearly
distinguish it from type parameters. This keyword is already used in a
different context. This use has a semantic relation to other uses of
`static` as a keyword. There is no ambiguity due to the context of generic
parameters in which it appears here either. Therefore it makes sense to use it
in this context.

Alternatively, one could also omit `static` or use a different keyword. Traits
would probably not be allowed as the type of a value parameter, since they
could not be statically resolved at compile-time. Therefore the parser should
be able to distinguish type parameters from static value parameters despite the
similarity, even without `static`. However, the difference between type
parameters and value parameters might become less obvious that way.

Here are a couple of alternatives:

```rust
fn add_n<n: int>(x: int) -> int { /* ... */ }
fn add_n<static n: int>(x: int) -> int { /* ... */ }
fn add_n<const n: int>(x: int) -> int { /* ... */ }
fn add_n<let n: int>(x: int) -> int { /* ... */ }
fn add_n<value n: int>(x: int) -> int { /* ... */ }
fn add_n<val n: int>(x: int) -> int { /* ... */ }

```

The `static` variant is arbitrarily chosen for the remainder of this document.


## Generic instantiation

When a generic item is instantiated, a concrete value needs to be statically
inferred. The simplest way to do this is to explicitly provide a static value:

```rust
let x = add_n<3>(2);

```

It may also be inferred from the context, as much as types can be inferred.
Consider this function returning a struct generic over some value parameter `n:
uint`:

```rust
struct Bar<static n: uint> { /* ... */ }
fn bar<static n: uint>(a: [f64, ..n]) -> Bar<n> { /* ... */ }

```

Given an array of statically known size `n`, it should be automatically
instantiated, so that the following type checks:

```rust
let x = bar([1.0, 2.4, -5.3]);

```

Here, `x` should have type `Bar<3>`.


## Default arguments

It is reasonable to specify a default value for a value parameter. This could
be expressed completely analogously to default type and lifetime parameters as
`"static" ident ':' type '=' expr`:

```rust
struct SmallVec<static size: uint = 16> {
    data: [T, .. size],
}

```


## Ordering of the kinds of parameters

When both value and type or lifetime parameters are present, the question
arises how they should be ordered. Lifetime parameters always come first in the
current system, thus it makes sense to place value parameters either between
lifetime and type parameters or at the very end. For this proposal the latter
rule is adopted.

Alternatively, one could also allow type and value parameters to be mixed.
This may be beneficial for API design as a grouping of parameters unrelated to
their kind could make sense. However, it would be inconsistent with the
previous restriction to have lifetime parameters come first.

```rust
fn foo<'a, T, static n: uint>(a: &'a [T,..n]) -> &'a T;

```

This also potentially has the advantage that the keyword could be omitted for
all but the first value parameter:

```rust
fn foo<T, U, static n: int, x: f32, b: bool>();

```


## Allowed types

The type of an instantiated generic must be known at compile time. This
naturally limits the types available for value parameters.

The most conservative approach would be to only allow `uint`. This is
effectively what C++ does and covers many use cases. However, there are still a
number of use cases that can not be addressed this way.

A generalization of this is to allow concrete sized types with a total
equality (those implementing `Eq`).

```rust
/// A function with some compile time flag
fn foo<static flag: bool>(a: int) -> int;

/// An integer within a range
struct Range<static lower: int, static upper: int> {
    n: int,
}

/// A range type, which may be bound
struct OptionRange<static lower: Option<int>, static upper: Option<int> {
    n: int,
}

```

There are a couple more possible generalizations. First of all, it might be
enough to expect the type of a value parameter to be `PartialEq`. This would
extend the eligible types to `f32`, `f64` etc. However, this means that there
are types, which can never successfully type check.

```rust
struct Foo<static x: f64>;

#[test]
fn nan_type() {
    // This is a type mismatch, since NaN != NaN
    let x: Foo<0.0 / 0.0> = Foo<0.0 / 0.0>;
}

```

To resolve this the compiler could only allow the subset of values of a type,
for which `x == x` holds, i.e. check this property for every value used to
instantiate a type.

Another issue is, whether types such as arbitrarily large integers and rational
numbers should be allowed.

Furthermore, it is conceivable to be generic over the type of a value
parameter, e.g.

```rust
/// A more generic range type
struct Range<T, static lower: T, static upper: T> {
    n: T,
}

```

A special trait may signal the compiler eligibility as a type for a value
parameter. This trait could be imposed in the above example as `T: Value`.


## Functions in arguments (CTFE)

A very diserable feature is to do some algebra with the parameters, like this:

```rust
fn concatenate<T, static n: uint, static m: uint>
    (x: Vector<T, n>, y: Vector<T, m>) -> Vector<T, n + m>
{
    let mut new_data: [T, ..n + m] = [0, ..n + m];
    for (i, &xx) in x.data.iter().enumerate() { new_data[i] = xx; }
    for (i, &yy) in y.data.iter().enumerate() { new_data[i + n] = yy; }
    Vector::new(new_data)
}

```

This feature should be restricted to a certain subset of functions, namely pure
functions without side-effects. Compile-time function execution (CTFE) as
tracked in [this RFC][ctfe_rfc] is required to instantiate the types
parametrized with value parameters that are the result of some function.
Describing CTFE is beyond the scope of this RFC but it certainly is required
for this particular feature. However, the feature can be omitted, as the rest
of this proposal works nicely without it.

Type inference can potentially become complex, especially when arbitrary
functions must be executed. It seems reasonable to restrict the use of
algebraic expressions in types in such a way, that it is still possible to
immediately infer a type from an expression without resorting to any kind of
function inversion.

Consider the following functions:

```rust
fn new<static n: int>() -> T<n>;
fn inc_explicit<static n: int>(x: T<n>) -> T<n + 1> {...}
fn inc_implicit<static n: int>(x: T<n - 1>) -> T<n> {...}

#[test]
fn test_inc() {
    // This should work with x: T<4>
    let x = inc_explicit(new<3>());
    // This can not be inferred
    let y = inc_implicit(new<3>());
    // However, with explicit annotations it should type check
    let z = inc_implicit<4>(new<3>());
}

```


## Traits and associated items

Given a complete implementation associated items and particularly associated
constants the canonical use cases for value parameters in generic traits can be
replaced by these concepts. Consider this example of a trait generic over value
parameters `p: P`, `q: Q`…:

```rust
trait Abstract<static p: P, static q: Q, ...> {
    /* ... */
}

struct Concrete<static p: P, static q: Q, ...> {
    /* ... */
}

impl<static p: P, static q: Q, ...> Abstract<p, q, ...> for Concrete<p, q, ...> {
    /* ... */
}

```

This can be rewritten like

```rust
trait Abstract {
    static p: P;
    static q: Q;
    // ...
    /* trait methods, other associated items */
}

struct Concrete<static p: P, static q: Q, ...> {
    /* fields */
}

impl<static p: P, static q: Q, ...> Abstract for Concrete<p, q, ...> {
    static p: P = p;
    static q: Q = q;
    // ...
    /* method impls, other associated items */
}

```

It is also possible to generically require a particular kind of `Abstract` in a
where-clause here:

```rust
fn foo<T: Abstract, static p: P, static q: Q, ...>() -> T
    where T::p == p, T::q == q, ...
{ /* ... */ }

```

There are examples though that would be impossible without value parameters in
traits, e.g.

```rust
struct Foo;

impl<static p: P, static q: Q, ...> Abstract<p, q, ...> for Foo {}

```

So for completeness this aspect should be included as well.


## Extended where clauses

In the last paragraph, where clauses comparing generic value parameters and
associated items were briefly mentioned. Given CTFE, another extension could be
to allow arbitrary boolean expressions containing value parameters as where
clauses.

An example:

```rust
/// Returns the middle element of a slice with odd length
fn center<T, static n: uint>(a: &[T, .. n]) -> T
    where n % 2 == 1
{ /* ... */ }

```

There are possible collisions:

```rust
impl<static n: uint> Foo<n> for Bar<n>
    where n % 2 == 1
{ /* ... */ }

impl Foo<3> for Bar<3>
{ /* ... */ }

```

How should this be resolved?


# Alternatives

Parts of the functionality provided by this change could be achieved using
macros, which is currently done in some libraries (see for example in
[Servo][servo_macros] or Sebastien Crozet's libraries [nalgebra][nalgebra] and
[nphysics][nphysics]. However, macros are fairly limited in this regard.


# Unresolved questions

- Which exact syntax should be used? (`static`, `const`, `value` etc.)
- In how far is compile-time function execution acceptable to support this?
- How does this proposal interact with other planned language concepts, such as
  higher-kinded types or variadic generics?
- A grammar specification is required


[servo_macros]: https://github.com/mozilla/servo/blob/b14b2eca372ea91dc40af66b1f8a9cd510c37abf/src/components/util/smallvec.rs#L475-L525
[nalgebra]: https://github.com/sebcrozet/nalgebra
[nphysics]: https://github.com/sebcrozet/nphysics
[issue_11621]: https://github.com/mozilla/rust/issues/11621
[ctfe_rfc]: https://github.com/rust-lang/rfcs/issues/322
[ctfe_mail]: https://mail.mozilla.org/pipermail/rust-dev/2014-January/008252.html
[template_meta]: http://en.wikipedia.org/wiki/Template_metaprogramming
[range_mod_ada]: http://en.wikipedia.org/wiki/Ada_%28programming_language%29#Data_types
[turing_complete]: http://www.reddit.com/r/rust/comments/2o6yp8/brainfck_in_rusts_type_system_aka_type_system_is/
