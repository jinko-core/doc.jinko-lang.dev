# WU0007: Partial Application operator

## Concept

It'd be nice for `jinko` to have an operator specific to the partial application of a function. Let's look at a few examples

## Why not reuse the call operator?

Reusing the call operator (`f(arg1, arg2)`) causes more issues than it solves in my opinion. It murks up the reading of code and makes reasoning about `jinko` semantics harder. Furthermore, typechecking errors now become confusing, and often, not errors on the call site!

Let's take the following example:

```rust
// TODO: Should be named `add3` really, but isn't...
// I hope this does not confuse users!
func add(x: int, y: int, z: int) -> int { x + y + z }

where x = add(1, 2);

println("{x}")
```

would produce something similar to the following output:

```rust
error: source.jk:5:11: cannot format expression of type `func(int) -> int`

  5 | println("{x}")
    |           ^

hint: no specialization exists for function `fmt` for type `func(int) -> int`

-> stdlib/fmt.jk:

 15 | func fmt[T](value: T) -> string; /* builtin */

hint: consider specializing the function

  5 | func fmt[fn(int) -> int](value: func(int) -> int) -> string { /* TODO */ }
```

As a side note, we could also consider using a similar syntax for [generic specialization](#generic-specialization).

## Design considerations

### Delimiting partial arguments?
### Application of multiple arguments at once
### Chaining applications

## Alternatives

Let's keep going with our weirdly named `add` function.

```rust
func add(x: int, y: int, z: int) -> int { x + y + x }
```

### `| arg0, arg1, arg2... |` syntax

```rust
where add1 = add |1|;
where res = add(2, 3); // typeck error
where res = add1(2);   // typeck error

where res = add1(2, 3); // 1 + 2 + 3, fully applied

where add1_2 = add |1, 2|;
where res = add1_2 |3|; // same result, partial application but "complete"
```


### `<- (arg0, arg1, arg2...)` syntax

```rust
where add1 = add <- 1;
where add1_2 = add <- (1, 2); // how to work with tuples?
where add1_2 = add1 <- 2;
```

### `<- arg0 <- arg1 <- arg2` syntax

```rust
where add1 = add <- 1;
where add1_2 = add <- 1 <- 2; // no problem with tuple arguments here
where add1_2 = add1 <- 2;
```

### `curry` function?

One could think about having a `curry` function turning a given function into its partial application

```rust
// simple case
func curry[T, Z](f: func(T) -> Z) -> func(T) -> Z {
    |t| f(t)
}

// two arguments
func curry[T, U, Z](f: func(T, U) -> Z) -> func(T) -> func(U) -> Z {
    |u| curry(|t| f(t))(u) // uuuuh, that's not valid
    // how?
}
```

Or have `curry` as an interpreter builtin? But then it contradicts with the goal of having the interpreter be as small
and simple as possible.

#### Generic Specialization

```rust
func fmt[T](value: T) -> string;

type Marker;

// ?
func fmt[T](value: T) -> string where T <- Marker {
    "Marker"
}

func fmt[T <- Marker](value: Marker) -> string {
    "Marker"
}

type Nothing;
type Maybe[T](inner: T | Nothing);

func fmt[T <- Maybe[T]](value: Maybe[T]) -> string {
    switch value.inner {
        contained: T => "{contained}",
        _: Nothing => "Nothing"
    }
}
```

This could also lead to partial application of generics!

```rust
func map[T, U](value: T, fn: func(T) -> U) -> U {
    fn(value)
}

func map[T <- Vector[T], U <- Vector[U]](vec: Vector[T], fn: func(T) -> U) -> Vector[U] {
    vec.iter().map(|value| map(value, fn)).to_vector()
}

// Transform a Maybe[int] into something or panic
func map[T <- Maybe[int], U](opt: Maybe[int], fn: func(int) -> U) -> U {
    switch opt {
        value: int => fn(value)
        _: Nothing => panic()
    }
}
```
