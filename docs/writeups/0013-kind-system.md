# WU0013: Kind system

## Reasoning

First class types are extremely useful, but do not play well with static typing - for example, what is the language supposed to do in cases such as:

```rust
func make_type(b: bool) -> type {
  if b {
    TypeBuilder.new().field("name", string)
  } else {
    TypeBUilder.new().field("a", int)
  }
}

type Foo = make_type(get_user_input_as_bool())

func say_name[T](value: T) {
  println(value.name)
}

Foo(a: 15).say_name()
```

If `false` is given as an argument to `make_type`, then the generated type will contain a field named "a" of type `int`. Otherwise, it will contain only one field "name" of type `string`. The second type is a valid type to the function `say_name`, but the first one isn't.
Allowing this behavior to error out at runtime rather than during typechecking is closer to dynamic typing, which `jinko` should not support. Thus, we must find a solution to restrain these "type generator" functions to compile time arguments.

I see two solutions to this problem:

1. Restrict arguments and return type of type generators to compile time values

2. Introduce a new `kind` syntax as a shorthand for a compile time function which returns a new type

## Proposal 1

We could have a magical `CompileTime[T]` type which would force its initializers to be compile time constants, e.g. by restricting the creation of a `CompileTime[T]` if `T` comes from an `Impure[T]`

Bikeshedding:

- [ ] `CompileTime[T]`
- [ ] `Known[T]`
- [ ] `Folded[T]`
- [ ] `Foldable[T]` -> sounds like something you could call `.fold()` on

- Since we are in an interpreted language, any pure function can be a compile-time argument, correct?
- We need an `Impure` type to mark functions which interact with the outside world

The syntax for type generators would then look something like this:

```rust
func make_type(b: Known[bool]) -> Known[type] {
  /* ... */
}

func extend(
  t: Known[type],
  field_name: Known[string],
  field_type: Known[type]
) -> Known[type] {
  /* ... */
}

type Foo = make_type(true);
type Bar = extend(Foo, "another_field", int);
```

The main issue with this is that we would be special casing functions returning `type`s anyway, so why require the addition of that type? We would need special handling for these anyway.

## Proposal 2

Add the notion of `kind` and a "kind system", which is a shorthand for functions generating new types at compile time.

A `kind` can be though of as a function which takes compile time constants as arguments and returns a new type.

```rust
kind MakeType[b: bool] {
  /* ... */
}

kind Extend[t: type, field_name: string, field_type: type] {
  /* ... */
}

type Foo = MakeType[true];
type Bar = Extend[Foo, "another_field", int];
```

This has the advantage that reusing the generic syntax makes it clear this is compile time only. Furthermore, it does not derive from the notion that `jinko` should only have three concepts - labels, types and functions. A `kind` is simply a sort of function which returns a new type.

## Examples

```rust
kind Extend[t: type, field_name: string, field_type: type] {
  where new_type = t.fields().fold(TypeBuilder.new(),
    (new_type, field) -> new_type.with(field)
  );

  new_type.with_field(TypeBuilder.Field(field_name, field_type))
}

type Point(x: int, y: int);
type Point3d = Extend[Point, "z", int];

kind Intersection[t: type, u: type] {
  where all_fields = t.fields().append(u.fields());

  where new_fields = all_fields.filter(|field| t.fields().contains(field) && u.fields().contains(field));

  TypeBuilder.new().fields(new_fields)
}

type Foo(a: int, b: int, c: string, d: int);
type Bar(b: string, c: string, d: float);
type Baz = Intersection[Foo, Bar]; // (c: string)

kind Source[path: string] { /* ... */ }

// Vectors of ZSTs are a little particular, in that they don't need to contain any allocation or capacity - simply a size.
// If the size is greater than zero, then they can simply return a copy of the ZST. Otherwise, they return nothing. Each
// `pop` decrements the size, and each `push` increments the size. This is a massive win for Vecs of marker types. We
// implement this distinction using a kind.
kind Vector[t: type] {
  where common_vec = TypeBuilder.new()
    .field("size", int)
    .field("inner_get", func[T](Vector[T], int) -> Maybe[T]);
  
  switch t {
    zst: Zst -> common_vec
      .field("data", zst),
    other: _ -> common_vec
      .field("data", RawPointer[other])
      .field("capacity", int),
  }
}

func inner_get_zst[T](v: Vector[T], idx: int) -> Maybe[T] {
  if v.size > 0 {
    v.data
  } else {
    Nothing
  }
}

func inner_get[T](v: Vector[T], idx: int) -> Maybe[T] {
  if i < v.size {
    v.data.offset_at(i * T.size())
  } else {
    Nothing
  }
}

func get[T](v: Vector[T], idx: int) -> Maybe[T] {
  Vector[T].inner_get(v, idx)
}
```

## Drawbacks

- [ ] How to handle errors? `kind`s returning an invalid type/error type?
- [ ] What does the type system look like for these?
- [ ] How does it interact with generics?
- [x] UFCS for kinds? -> No
