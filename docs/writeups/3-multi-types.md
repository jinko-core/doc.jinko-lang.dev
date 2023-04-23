# Sum types in jinko

`jinko` has sum types which can be seen as an abstraction over a choice of multiple types at once.
They are very useful when you are trying to represent data which might exist in multiple states, such as a valid state and an invalid one.

This pattern is common in functional programming languages such as Rust or Haskell. In Rust, a sum type you'll often see used is the `Option`
enum, whose definition looks something like this:

```rust
enum Option<T> {
    Some(T),
    None,
}
```

`jinko`'s sum types are similar to Rust's, with one key difference: in Rust, the variants of a sum type can only be used as enum variants - not as types or data instances themselves. They are a proxy to the information they contain. For example in Rust, you cannot pass a value of type `Some<i64>` to a function. Nor can you give it a `None`:

```rust
let a = Some(15); // we *know* this is the Some variant of an Option - not None

fn foo(value: Some<i32> /* Invalid */) {}
fn bar(value: Option<i32>) {} /* Ok */

bar(a);
```

What you can do however, is give the information contained within those variants: in our case, either an integer, or... nothing:

```rust
let a = Some(15);

fn works_on_some(value: i32) {}
fn works_on_none(/* nothing */) {}

match a {
    Some(inner) => works_on_some(inner),
    None => works_on_none(),
} // Ok
```

This is fine for most cases, but gets annoying when creating your own sum types. Let's imagine we are trying to abstract types with lots of information, and want to represent the visitors of a zoo. This zoo accepts adults, children, assistance pets.

```rust
// a visitor can be a person, a child or an assistance pet
enum Visitor {
    Person,
    Child,
    AssistancePet,
}
```

This is all fine and dandy, now let's add information to each of our variants:

```rust
enum AssistancePetKind {
    Dog,
    Cat,
    Poney,
    Rabbit,
}

enum Visitor {
    Person {
        name: String,
        last_name: String,
        age: u64,
    },
    Child {
        /* children are anonymous in this zoo */
        age: u64,
        favorite_animal: String,
        number_of_ice_cream_cones_consumed: u64,
    },
    AssistancePet {
        kind: AssistancePetKind,
        name: String,
        harness: bool,
    },
}
```

if we were to have a function for each of our variants, the code would quickly get out of hand:

```rust
fn handle_person(name: &str, last_name: &str, age: u64) {}
fn handle_child(age: u64, favorite_animal: &str, ice_cream_cones: u64) {}
fn handle_assistance_pet(kind: &AssistancePetKind, name: &str, harness: bool) {}

match visitor /* of type Visitor */ {
    Visitor::Person {
        name,
        last_name,
        age,
    } => handle_person(name, last_name, age),
    Visitor::Child {
        age,
        favorite_animal,
        number_of_ice_cream_cones_consumed,
    } => handle_child(age, favorite_animal, number_of_ice_cream_cones_consumed),
    Visitor::AssistancePet {
        kind,
        name,
        harness,
    } => handle_assistance_pet(kind, name, harness),
}
```

There is a lot of repetition, and if we were to add more fields to our variants, we would need to change our function's declaration, our match pattern, as well as our match expression for that pattern. Furthermore, having too many arguments to a function hinders readability and causes a higher mental load when using said function.

We can thus wrap our data in new external types to deal with:

```rust
struct Person {
    name: String,
    last_name: String,
    age: u64,
}

struct Child {
    /* children are anonymous in this zoo */
    age: u64,
    favorite_animal: String,
    number_of_ice_cream_cones_consumed: u64,
}

struct AssistancePet {
    kind: AssistancePetKind,
    name: String,
    harness: bool,
}

enum AssistancePetKind {
    Dog,
    Cat,
    Poney,
    Rabbit,
}

enum Visitor {
    Person(Person),
    Child(Child),
    AssistancePet(AssistancePet),
}

fn handle_person(person: &Person) {}
fn handle_child(child: &Child) {}
fn handle_assistance_pet(pet: &AssistancePet) {}

match visitor /* of type Visitor */ {
    Visitor::Person(p) => handle_person(p),
    Visitor::Child(c) => handle_child(c),
    Visitor::AssistancePet(pet) => handle_assistance_pet(pet),
}
```

This is better, but we still have a lot of repetition. For each variant in our `Visitor` enum, we need to write the type of data we are keeping *twice*. Each variant cannot stand on its own as a meaningful value. This leads to some confusion but most importantly repetition.

Let's look at the equivalent `jinko` code.

```rust
type Person(
    name: string,
    last_name: string,
    age: int,
);

type Child(
    /* children are anonymous in this zoo */
    age: int,
    favorite_animal: string,
    number_of_ice_cream_cones_consumed: int,
);

type Dog;
type Cat;
type Poney;
type Rabbit;

type AssistancePet(
    kind: Dog | Cat | Poney | Rabbit,
    name: string,
    harness: bool,
);

type Visitor = Person | Child | AssistancePet;

func handle_person(person: Person) {}
func handle_child(child: Child) {}
func handle_assistance_pet(pet: AssistancePet) {}

switch visitor /* of type Visitor */ {
    p: Person -> handle_person(p),
    c: Child -> handle_child(c),
    pet: AssistancePet => handle_assistance_pet(pet),
}
```

There is less repetition, and it is easier to add fields or variants to your types, as well as handle them later in your code. Let's have a look at their implementation as well as some of the design choices behind them.

```rust
type Red;
type Blue;
type Green(is_light: bool);

type Color = Red | Green | Blue;

where color: Color = Green(is_light: true);
```

we instantiate a value of type `Green`, but assign it the type `Color`. This is fine and allowed, since the type `Green` is part of the multitype `Color`.

On the other hand, this fails to typecheck:

```rust
where color: Red | Blue = Green(is_light: true);
```

## Primitive types

`jinko` has 5 primitive types, which are all sum-types. The easiest one to understand is `bool`, which simply represents either an instance of `true` or an instance of `false`.

```rust
type false;
type true;
type bool = false | true;
```

The `char` type can be seen in the same way - it is a sum type, which englobes all possible characters representable in `jinko`: 'a', 'b', 'c'... up until the last valid Unicode codepoint.
This would take a lot of space to represent directly at the source level, so that type is implemented directly in the compiler. We use a similar technique for `int`, `float` and `string`.

What this means is that each character, integer, floating-point number or string is a *type* in `jinko`.

```rust
where x: 15 = 15; // x is a value of type 15, whose value is 15
where greet: "hey" = "hey";
where greet2: "hey" = "hello"; // type error
```

On their own, these types are not very useful. They become interesting when we try and restrict the data our functions or types accept. We can imagine the following snippet, which goes in a cardinal direction based on a string input:

```rust
func advance_cardinal_direction(s: string) -> Result {
    switch s {
        "north" -> go_north(),
        "south" -> go_south(),
        "west" -> go_west(),
        "east" -> go_east(),
        _ -> Error(from: "invalid cardinal direction: {s}" ),
    }
}
```

or, we could also restrict this function to only accept valid cardinal directions at compile time:

```rust
// no need to return a Result - this function does not typecheck
// if it does not have a proper cardinal direction as an argument
func advance_cardinal_direction(s: "north" | "south" | "west" | "east") {
    switch s {
        "north" -> go_north(),
        "south" -> go_south(),
        "west" -> go_west(),
        "east" -> go_east(),
        // no error branch
    }
}
```

This transfers the responsibility to your library's users to sanitize their input and send your function proper datas. The same principle applies to types.

## Design

```rust
where x = if condition { 15 } else { "jinko" };
// x has type `int | string`

where mut x = 15;
x = "jinko";
// type error
// but the typechecker hints that you can specify x's type to work around this:

where mut x: int | string = 15;
x = "jinko";
```

## Looking at a few common types

```rust
type Nothing;
type Maybe[T] = T | Nothing;

func map[T, U](value: Maybe[T], mapper: func(T) -> U) -> Maybe[U] {
    switch value {
        inner: T -> mapper(inner),
        Nothing -> Nothing,
    }
}

func extract_or[T](value: Maybe[T], default: func() -> T) -> T {
    switch value {
        inner: T -> inner,
        Nothing -> default(),
    }
}

func checked_div(lhs: int, rhs: int) -> Maybe[int] {
    switch rhs {
        0 -> Nothing,
        1 -> lhs,
        _ -> lhs / rhs
    }
}

func is_whitespace(c: char) -> bool {
    switch c {
        '\t' | ' ' -> true,
        weird: '\r' | weird: '\n' -> {
            eprintln("{weird} is one but it's weird right?");
            true
        },
        rest: _ -> {
            eprintln("got a non-whitespace char: {rest}");
            false,
        }
    }
}
```

```rust
switch_case := 'switch' expr '{' [ pattern '->' expr ]+ '}'
pattern     := ( pat_type | identifier ':' pat_type ) [ '|' pattern ]*
pat_type    := '_' | type
```

## Limitations

1. Typechecking error when using `switch` on a float

How to work around this? Have a safe float type which cannot be NaN and has proper epsilon-aware comparisons?

2. Switching on a value of type `All/Any`?

## Pending questions

### Nested pattern syntax

How to handle it? This is a fantastic feature, and we want it.

## Compilation/Desugaring

Sum types can be seen as tagged unions for the most part. A "tag", often a byte or small integer value, indicates the variant we are dealing with, and enough memory is allocated to hold an instance of one of the types constituting the sum type.

Let's look at the following `jinko` sum type:

```rust
type Point(x: int, y: int);
type Point3D(x: int, y: int, z: int);

type Both = Point | Point3D;
```

The `Both` sumtype could be "compiled" to something like this in a C-like language:

```c
struct Point {
    int x;
    int y;
};

struct Point3D {
    int x;
    int y;
    int z;
};

enum Both_tag {
    BOTH_POINT,
    BOTH_POINT3D,
};

union Both_data, {
    Point,
    Point3D,
};

struct Both {
    Both_tag tag;
    Both_data data;
};
```

We can perform this desugaring directly in `jinko` code. Let's look at the colors example again.

```rust
where (color_type_mark, color_value) = (1, Green(is_light: true));

// then, when matching on `color`
switch color {
    _: Red -> {},
    _: Blue -> {},
    _: Green -> {},
}

// could get compiled/lowered to
switch color_type_mark {
    0: Red -> { /* we use color_value */ },
    1: Blue -> { /* we use color_value */ },
    2: Green -> { /* we use color_value */ },
}
```

Likewise for anonymous multi-types:

```rust
func takes_red_or_blue(color: Red | Blue, some_arg: int) {
    switch color {
        /* ... */
    }
}

// becomes
func takes_red_or_blue(color_type_mark: int, color_value: Red | Blue, some_arg: int) {
    switch color_type_mark {
        /* ... */
    }
}
```

Primitive sum types such as `char`, `string` and `int` are special in that they contain an
extremely high number of variants which are handled internally by the compiler. Having a
tag for a jinko `int` would not make sense, as we would need to keep track of `u64::MAX`
variants for the tag enum, as well as the integer it represents. This would effectively make
the representation of a 64 bits integer take at least 128 bits in memory, for no good reason
at all.
We can instead make a special case for the int type and directly use its value as the tag. We
can do a similar optimization for `char` values, which can be seen as small integer (a unicode
codepoint takes 4 bytes at most).

There is sadly an even higher number of possibilities for strings, as they can take up to
`u64::MAX` bytes of space on most 64-bits platforms. There are 1,114,112 y possible values for
a unicode codepoint, so there are literally billions of possible strings - which all need to
be contained in the `string` sum type! This is not representable using a regular C enum, or
any usual tag of some sort. Instead, we can simply compare the hashes of strings, which ends
up being a "simple" integer comparison. We will obviously need to think about collisions, but
this is a much more manageable problem.
