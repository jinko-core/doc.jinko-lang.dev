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

## An interpreter of type variables

### Type widening

Process of replacing the type of a variable with a sum type containing its original type.

e.g. for a variable of type `A | B | C`, assign it the type `A | B | C | D | E`.

In set terms: we are replacing a given set `E`, by a set `F` containing all the elements of `E` and one or more elements.

```rust
func foo(
  a: int | string // :: type { int, string }
) {}

foo(15); // 15 :: type { int }
foo("jinko"); // "15" :: type { string }
```

Both the expressions above are correct and typecheck properly. In both cases, the value given to `foo` will see its type "widened" to the type of `a`: `int | string`. Type widening does not occur if the expression is already of the expected type, etc:

```rust
where x: int | string = // :: type { int, string }
    if condition() {
        15 // :: type { int }
    } else {
        "jinko" // :: type { string }
    };

foo(x); // x :: type { int, string }
```

In the last call to `foo`, no widening needs to happen - both the set of `x` and the set of `a` are already equal to one another.

### Type narrowing, or flow-sensitive typing

Flow-sensitive typing is the process of "narrowing down" (as opposed to "type widening") the sum type of an expression to one variant of that sum type.

e.g. for a variable of type `A | B | C | D`, allow narrowing it down to a variable of type `A`, OR one variable of type `B`, OR one variable of type `C | D`.

_Flow-sensitive typing can only happen in switch expressions_.

This is an important distinction from other languages with flow-sensitive typing. `jinko` does not allow flow-typing in `if-else` expressions, and all flow-typing must be exhaustive.

Reduce a given set `E` of `n` elements to `n` or less sets containing possibly only one element.

### Set expansion

### Representation of type variables

Type variables need to be represented in a way that is simple to understand, and simple to reason about. The easiest way is to represent them as a set of type nodes - basically a set of indexes into the `Fir`.
This works well for simple types, but completely falls apart for our "magic"
builtin primitive union types: `int`, `char` and `string`. While it is possible
to represent them as sets of all possible integer, character and string
literals, it does not make sense to do so and will certainly incur a heavy
compilation penalty. We can then think about representing our type variables like this:

```rust
enum Type {
    /// A primitive union type: int, char or string. The index corresponds to the definition of
    /// that type in the standard library.
    PrimitiveUnion(OriginIdx),
    /// A "regular" type. The index corresponds to the definition, and the typeset to the set of 
    /// types represented by that definition.
    Set(OriginIdx, TypeSet),
}
```

For our subtyping rules, we can simply check if a given typeset is contained in the expected typeset. But this does not work for primitive type unions: We do not know in advance the indexes of all constants used in a program, and cannot easily know if a constant's type index is present in a union's type index: Let's look at this with the following program.

```rust
type char;
type int;
type string;

where x = 15;
```

Since `x` will be of type `15`, it's as if we had an extra type declaration in the program:


```rust
type char;
type int;
type string;

type 15;

where x = 15;
```

Now let's add type variables to all our expressions. Remember that a record type will show up as a typeset of one member, itself.

```rust
// Type::PrimitiveUnion(1)
type char;
// Type::PrimitiveUnion(2)
type int;
// Type::PrimitiveUnion(3)
type string;

// Type::Set(4, { 4 });
type 15;

// Type::Set(4, { 4 }) // same type as `type 15`
where x = 15;
```

There is absolutely zero link between the type of the constant `15` and the union type `int`. Checking if the set of `15` fits within the set of `int` does not make sense, since the set of `int` is empty.

There are multiple ways to solve this problem:

1. When doing typeset comparisons, fetch node information from the `Fir`. This allows us to look at a node's AST data, and to check if it is an `int` constant or not - if that is the case, then that node can be widened to a node of type `int`.
2. When typing constant nodes, add extra information regarding the primitive type they could widen to. Basically storing something like `Type::ConstantSet(4, can_widen_to: "int")` in the above example. This works but is delicate, spaghetti, and causes us to add an extra variant to our enum
3. Collect a list of all the constants in a program in order to actually create a primitive type that is a proper union type. This would allow us to *remove* a variant from our enum, as `int` would simply become a union type with an actual type set.

```rust
// Type::Set(1, {})
type char;
// Type::Set(2, { 4 })
type int;
// Type::Set(3, {})
type string;

// Type::Set(4, { 4 });
type 15;

// Type::Set(4, { 4 }) // same type as `type 15`
where x = 15;
```

This would work and also be quite simple to implement.
