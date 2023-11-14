# WU0014: Type system

This writeup details `jinko`'s type system, based on a primitive kind system. It does not *need* a kind system to exist, but the kind system is a way to extend the type system further.

Recommended reading: [Haskell's kind system - a primer](https://diogocastro.com/blog/2018/10/17/haskells-kind-system-a-primer/)

## Kinds

`kind`s are a way to describe types and type constructors within `jinko`'s type system. Let's take a few examples to illustrate our problem:

```rust
type Point(x: int, y: int);
```

the type `Point` is of kind `type`. It is a fully formed type, which you can instantiate directly - similarly to `int` or `string`:

```rust
where x: int = 15;
where y: string = "foo";
where p: Point = Point(x: 1, y: 2);
```

In the Haskell world, we would say that `Point` is of kind `*`, which we will call `type` here.

On the other hand, let's consider a type with a generic argument:
```rust
type PairSame[T](fst: T, snd: T);
```

`PairSame[T]` is not a fully fledged type - you cannot instantiate a variable of type `Point[T]` - `T` must be known:

```rust
where p: PairSame[T] = PairSame[T](fst: ???, snd: ???); // invalid
where p: PairSame[int] = PairSame[int](fst: 14, snd: 15);
where p: PairSame[string] = PairSame[string](fst: "str", snd: "ing");
```

Hence why you could see `Point[T]` as a "type constructor". It takes an extra type `T` as an argument, and "creates" a fully fledged type through monomorphization.
You can think of `Point[T]` as a sort of *function* which will take a type as its only argument, and return a new type.

In Haskell, `Point[T]` would be of kind `* -> *`: from a fully fledged type of kind `*`, create a new fully fledged type of kind `*`.
We will say that `Point[T]` has kind `type -> type`.

Even more complicated: `jinko`'s module system creates a new `type` out of the path of a module, encoded as a string. Out of a `string`, create a new `type`. This has kind `string -> type`.
If we imagine a "thing" which requires an integer and a type to return a new type, then that "thing" would be of kind `(int, type) -> type`.

## Creating types

With this knowledge in mind, `jinko`'s type system can be thought of as an interpreter which only deals with kinds and which has no side-effects. All of the execution and logic happens during type-checking.
In that system, a concrete type of kind `type` can be thought of as a regular immutable variable/binding. A type constructor of kind `type -> type` can be thought of as a regular function.

In the following code examples, comments will be added around the various types to show the kinds we are dealing with:

```rust
type Foo; // :: type

type Foo[T]; // :: type -> type
```

There are two different types in `jinko`: sum types and record types. Sum types are made of multiple possible types but instances of a sum type are only ever *one* type. Record types on the other hand, are a product type: they contain multiple instances of possibly multiple types.

```rust
type Foo;
type Bar(a: int, b: Foo);
```

In the above example, `Foo` is a record type with no data fields. `Bar` is a record type with two fields: `a`, of type `int`, and `b`, of type `Foo`.

```rust
func foo(a: int | bool | char) {}
```

Here on the other hand, the argument `a` is of the sum type `int | bool | char`. `a` will be either an integer, a boolean or a character. Let's add the kinds to these two examples:

```rust
type Foo; // :: type
type Bar(
  a: int, // :: type
  b: Foo, // :: type
); // :: type

func foo(
  a: int | bool | char // :: type
) {}
```

We can extract a few things from these examples. The `type` keyword can be used to create new types, of kind `type` - new, fully fledged types. We can also use it to create type aliases:

```rust
type Foo; // :: type
type Bar = Foo; // :: type
```

This is similar to creating two "variables" in a regular piece of code. One is named `Foo`, and the other, `Bar`, refers to `Foo`. `Bar` is a type alias of `Foo` - not a new type. Anywhere `Foo` is used, `Bar` could be used, and vice-versa.

Another concept we introduce in the above example is that function arguments can be typed with values of kind `type`. It is not possible for a function argument to be of kind `type -> type`, or other type constructors. This is an important concept, as one could think tihs is not the case with the following example:

```rust
func foo[T](value: Maybe[T]) {}
```

However, this function is only valid when fully monomorphized - meaning that `value` will only ever accept a type of kind `type`.

## Sets

In `jinko`, types can all be represented as sets. According to wikipedia, sets are ["the mathematical model for a collection of different things"](https://en.wikipedia.org/wiki/Set_(mathematics)), so we will call the components of our sets "things".

Thus, in `jinko`, a type is a `Set` of `Thing`s. The notation we will use in the examples is as follows:

1. Each `Thing` will be written as its name: `int`, `Complex`, `float`
2. Each type will be represented as the set of its things using curly brackets: `{ int, string }`

```rust
where x /* { int } */ = 15;

where y: string | int /* { string, int } */ =
  if condition() { x } else { "none" }
```

We can add a number of rules around sets and things:

1. The type of a binding is always a `Set`, even if it contains only one `Thing`

```rust
where x = 196.4; // { float }

func foo(
    a: int,  /* { int } */
    b: bool, /* { bool } */
) -> char    /* { char } */ {
    'a'      /* { char } */
}            /* { func({int}, {bool}) -> {char} } */
```

2. `switch` expressions are the only ones allowed to explore the `Thing`s within a `Set`

```rust
where x = int | string | char = '1'; // { int, string, char }

switch x {
    i: int -> handle_int(i       /* { int } */),
    s: string -> handle_string(s /* { string } */),
    c: char -> handle_char(c     /* { char } */),
}
```
