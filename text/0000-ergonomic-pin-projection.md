- Feature Name: ergonomic-pin-projection
- Start Datae: 2022-09-08
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Pin projections are currently only achievable via `unsafe`. This RFC proposes to
enable users to access the fields through `Pin<&mut T>`:
```rust
struct Foo {
	a: usize,
	#[pin]
	b: PhantomPinned,
}

fn project(foo: Pin<&mut Foo>) {
	let _: &mut usize = &mut foo.a;
	let _: Pin<&mut PhantomPinned> = &mut foo.b;
}
```

# Motivation
[motivation]: #motivation


Pin projections have to be used when manually designing futures. They also arise
when having to handle unmovable types from C in the linux kernel.
While the crates [`pin-project`] and
[`pin-project-lite`] already exist and
solve this issue, they still require the use of a macro and are not a first
party solution to the problem.

[`pin-project`]: https://crates.io/crate/pin-project
[`pin-project-lite`]: https://crates.io/crate/pin-project-lite



# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation


When using `Pin<&mut T>` you can access the fields as you normally would when
using `&mut T`. The difference is, if at the struct declaration you specifed
`#[pin]` on the field, then the pointer you receive from doing `&mut t.x`
will be `Pin<&mut X>`. Here is an example to illustrate how this works:
```rust
pub struct Foo {
	a: usize,
	b: u64,
	#[pin]
	c: YieldFuture,
}

fn project(foo: Pin<&mut Foo>) {
	let _: &mut usize = &mut foo.a;
	let _: &mut u64 = &mut foo.b;
	let _: Pin<&mut YieldFuture> = &mut foo.c;
}
```
Creating a `Pin<&mut Field>` or `&mut Field` from `Pin<&mut Struct>` is called a
pin-projection.

Another important difference is that pinned fields cannot be moved out of the
reference:
```rust
pub struct Foo {
	#[pin]
	fut: YieldFuture,
}

fn project(foo: Pin<&mut Foo>) {
	let _: YieldFuture = foo.fut;
	//                       ^^^ cannot move out of pinned reference
	//                       help: borrow the field `&mut`.
}
```

You can also chain these expressions to access fields deep within a pinned struct:
```rust
struct Outer {
	#[pin]
	inner: Inner,
	count: usize
}

struct Inner {
	#[pin]
	fut: YieldFuture,
	#[pin]
	foo: Foo,
	bar: Bar,
}

struct Foo {
	#[pin]
	fut: YieldFuture,
	#[pin]
	bar: Bar,
}

struct Bar {
	#[pin]
	fut: YieldFuture,
}

fn project(outer: Pin<&mut Outer>) {
	let _: Pin<&mut YieldFuture> = &mut outer.inner.foo.bar.fut;
}
```



# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

## Struct declaration
[struct-declaration]: #struct-declaration

When declaring a struct/enum, authors can add the marker attribute `#[pin]`
on each field they wish to have structurally pinned. If the type of the field is
`Unpin`, a warning is emitted:
```rust
struct Foo {
    #[pin]
//  ^^^^^^ cannot pin a field that is `Unpin`
//         help: remove this `#[pin]`
//         note: Pinning a type that is `Unpin` has no effect, as it can be
//               unpinned at any time. See [here](TODO: rustdoc link) for more.
	a: usize,

}
```

## Unpin interaction
[unpin-interaction]: #unpin-interaction

Unpin can now be implemented more acurately, because all of the structurally
pinned fields are known. Unpin will be automatically implemented for all types
that only structurally pin `Unpin` fields.

## Drop implementation
[drop-implementation]: #drop-implementation

Because the drop implementation also is not allowed to move out of the fields,
this could pose a problem. One way to fix this would be to change the signature
of drop to be `fn drop(self: Pin<&mut Self>)`. Types that have no structurally
pinned fields would not need to change anything, as `self.field` would resolve
to the same thing as before.

# Drawbacks
[drawbacks]: #drawbacks

- there already exists a macro solution
- the new `Unpin` [implementing behaviour][unpin-interaction] is a breaking
  change, because types that are currently using [`pin-project`] would implement
  `Unpin` despite allowing pinned references to `!Unpin` fields.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

- Why is this design the best in the space of possible designs?
- What other designs have been considered and what is the rationale for not choosing them?
- What is the impact of not doing this?

# Prior art
[prior-art]: #prior-art

Discuss prior art, both the good and the bad, in relation to this proposal.
A few examples of what this can include are:

- For language, library, cargo, tools, and compiler proposals: Does this feature exist in other programming languages and what experience have their community had?
- For community proposals: Is this done by some other community and what were their experiences with it?
- For other teams: What lessons can we learn from what other communities have done here?
- Papers: Are there any published papers or great posts that discuss this? If you have some relevant papers to refer to, this can serve as a more detailed theoretical background.

This section is intended to encourage you as an author to think about the lessons from other languages, provide readers of your RFC with a fuller picture.
If there is no prior art, that is fine - your ideas are interesting to us whether they are brand new or if it is an adaptation from other languages.

Note that while precedent set by other languages is some motivation, it does not on its own motivate an RFC.
Please also take into consideration that rust sometimes intentionally diverges from common language features.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- what kind of attribute would `#[pin]` be? would it be compatible with
  [`pin-project`]?

# Future possibilities
[future-possibilities]: #future-possibilities

- adding support for projecting uninitialized data:
```rust
let x: Pin<&mut MaybeUninit<X>>;
let _: Pin<&mut MaybeUninit<Foo> = x.foo;
```
