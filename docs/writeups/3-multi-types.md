# Sum types in jinko

`jinko` has sum types or multi types, which can be seen as 

## deciding on a keyword
## deciding on a syntax for patterns

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

```rust
// can get compiled/lowered to 
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

__FIXME__: Annoying to write types we don't map to values. How do we make that syntax better?

```rust
switch_case := 'switch' expr '{' [ pattern '->' expr ]+ '}'
pattern     := ( pat_type | identifier ':' pat_type ) [ '|' pattern ]*
pat_type    := '_' | type
```
