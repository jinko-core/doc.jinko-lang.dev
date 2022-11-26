# First class types and UFCS Generics

## Overall

```rust
func add(lhs: int, rhs: int) -> int { lhs + rhs }

add(1, 2)
1.add(2)

func id<T>(value: T) -> T { value }

id.<int>(15);

15.id.<int>();
<int>id(15);

type Tuple<T, U>(f: T, s: U);

t = Tuple.<bool, string>(f: false, s: "hey");

func takes_tuple(t: Tuple<bool, string>) -> bool { t.f }
```

## Generics declarations

They don't change!

```rust
type Generic<T>(inner: T);
