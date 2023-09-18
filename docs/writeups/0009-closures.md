# WU0009: Closures

## Syntax

```rust
// look at previous writeup regarding switches?
pattern := identifier | '(' pattern [ ',' pattern ]* ')'
closure := pattern '->' expr
```

## Tuples as argument lists

## How to pass closures as arguments?
## How should function arguments look like?

```rust
func map[T, U](value: Maybe[T], f: Func[(T), U]) -> Maybe[U] {
    switch value {
        Nothing => Nothing,
        inner: T => f(inner),
    }
}
```

## How to capture variables?

### Issues?
### design consideration?
### generalize to functions

## Code examples

```rust
where values = [1, 2, 3];

values
  .map(elt -> elt * 2)
  .zip()
  .for_each((i, elt) -> println("value at {i}: {elt}"))
  // or
  .for_each(tup -> println("value at {tup[0]}: {tup[1]}"))
```

