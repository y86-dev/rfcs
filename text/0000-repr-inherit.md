- Feature Name: `repr-inherit`
- Start Date: 2022-05-13
- RFC PR: [rust-lang/rfcs#0000](https://github.com/rust-lang/rfcs/pull/0000)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Introduce a new `repr` variant: `#[repr(inherit($type))]`.

When a type `A` is marked with this `#[repr(inherit(B))]`, the compiler ensures that `A` and `B` have the same layout.

# Motivation
[motivation]: #motivation

`#[repr(inherit($type))]` allows transmuting between two types with transmutable fields without requiring the use of `#[repr(C)]`.
This allows the compiler to still choose the optimal layout, while providing simple a way to have predictable layouts.

## Examples

```rust
struct FooUninit {
	pub a: MaybeUninit<u8>,
	pub b: MaybeUninit<u32>,
}

#[repr(inherit(FooUninit))]
struct FooInit {
	b: u32,
	a: u8,
}
```
Without `repr(inherit)`, both structs would need `#[repr(C)]` and we would have to list the fields in the same order.
When many fields and structs are involved (possibly with `#[cfg(...)]` enabling/disabling fields) this is error-prone.
It also requires that the programmer chooses a good order of the fields to avoid padding bytes.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

## `repr(inherit($type))`

This repr attribute ensures that the type the attribute is found on has the same in-memory representation as the type given in the attribute. This means that it is sound to `mem::transmute` the two types to and from each other (you still need to ensure custom invariants are upheld).
The two types of course need to have the same size and specify the same name for each field. The order of the fields may be different but the types need to be layout compatible.
For example:
```rust
struct FooUninit {
	pub a: MaybeUninit<u8>,
	pub b: MaybeUninit<u32>,
}

#[repr(inherit(FooUninit))]
struct FooInit {
	b: u32,
	a: u8,
}
```
The layout of `Foo{Uninit, Init}` is `repr(Rust)` (implicit). It will probably be laid out in a way that avoids unnecessary padding bytes and the `u32` will come first.

This repr allows you to write layout dependent types that still can have memory layout optimizations handled by the compiler.

The fields of the two types need to be layout compatible. This means that they are
- the same type
- two types `A` and `B` where `A` has the transitive `#[repr(inherit(B))]` attribute
- two types `A` and `B` where `A` has the transitive `#[repr(transparent)]` attribute and contains only one field of type `B`

"transitive" in this context means, that the attribute behaves transitively, so the following is legal:
```rust
struct A {
	// ..
}

#[repr(transparent)]
struct B {
	a: A,
}

#[repr(transparent)]
struct C {
	b: B,
}

#[repr(transparent)]
struct D {
	pub c: C,
}

#[repr(inherit(D))]
struct E {
	c: A,
}
```

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

I have no knowledge of how the compiler computes the layout of a type, but I have seen some parts of the miri codebase allocating layout for types (I believe these are the same).
I hope that the compiler has a bit more information than just the alignment and size of a type, otherwise we would need to add that.

## Compiler changes

In order to provide the feature, the compiler needs to
1. compute the layout of all types not marked with/without transitive fields with `#[repr(inherit(..))]`
2. iteratively compute the layout of all types marked with `#[repr(inherit(..))]`
  - check that the fields with the same name have compatible layouts and that all fields are present
  - check that no conflicting `#[repr(inherit(..))]` was specified

There should exist an iteration limit, for the beginning we could choose something low like 16. There should also exist a similar option to `#[recursion_limit()]`, say `#[repr_inherit_iter_limit()]` to select the limit.

## Semver

The attribute is only allowed to reference types with all fields visible to the current module and incompatible with types marked by `#[non_exhaustive]`. This applies to only the type mentioned in the parenthesis, so the struct you define can have private fields. This is because the attribute relies upon types not renaming fields, changing field types and adding types for semantic version compatibility.

# Drawbacks
[drawbacks]: #drawbacks

Why should we *not* do this?
- it complicates (more code to maintain, longer compile times) the layout selection process of the compiler
- adds more difficulty trying to learn rust layouting (should be marginal)

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

## `Repr` trait instead of `#[repr]`

Alternatively we could add a trait to use instead of `#[repr]` attributes altogether:
```rust
pub trait Repr {
	type Layout: sealed::LayoutType;

	const ALIGN: Option<usize>;
	const PACKED: Option<usize>;
}

// all implement sealed::LayoutType
pub mod layout {
	pub struct Rust;
	pub struct C;
	pub struct Transparent;
	pub struct U8;// etc.
	pub struct Inherit<T>(PhantomData<T>);
}
```
`#[repr(...)]` would then become an implementation of the trait.

## What if this is not implemented
Functionality wise, this can already able done, if one is careful. Using `#[repr(C)]` and ensuring the same order of fields and transmutibility between them also results in transmute compatible layouts. However we are losing
- automatic (and future) layout optimizations
- layout randomizations of `repr(Rust)`, helping with security
- compiler assisted layout generation - we are essentially doing the layouting manually (which is error prone and a burden)

# Prior art
[prior-art]: #prior-art

I (y86-dev) have not been able to find any discussions on this topic on zulip or the internals forum. There are however related topics:
- https://internals.rust-lang.org/t/specifying-a-set-of-transmutes-from-struct-t-to-struct-u-which-are-not-ub/10917
- https://github.com/rust-lang/rfcs/blob/master/text/2835-project-safe-transmute.md

# Unresolved questions
[unresolved-questions]: #unresolved-questions

- How does the compiler layout mechanism need to be adjusted?
- Do we need more complex rules? I wanted to keep it simple.
- what about the primitive reprs like `#[repr(u32)]`?

# Future possibilities
[future-possibilities]: #future-possibilities


## Project safe transmute
Integration with [project-safe-transmute](https://github.com/rust-lang/rfcs/blob/master/text/2835-project-safe-transmute.md) seems to be a good idea, types with `repr(inherit(...))` should be soundly transmutable.

## Enums and Unions
Allow the repr to be used on other types as well. I do not know how useful this would be, but I think in theory it could be done.

