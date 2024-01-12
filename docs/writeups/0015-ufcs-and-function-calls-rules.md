# WU0015: UFCS and function resolution rules

## Issue

We want the language to have UFCS in order to allow "piping" of operations without a dedicated piping operator or object system.

Therefore, we would like to reuse the "field access operator" (`.`) in order to perform calls as if they were "method calls". The simplest form of UFCS is as follows:

```rust
12.add(15)

// desugars to
add(12, 15)
```

This is well and good and already implemented in the interpreter. However, the situation gets trickier with modules/multiple compilation units.

Let's take the following `pair` module, which we'll use for the remainder of the writeup.

```rust
// pair.jk

type Pair[T](fst: T, snd: T);

func pair[T](fst: T, snd: T) -> Pair[T] { Pair(fst: fst, snd: snd) }

func map[T](p: Pair[T], f: T -> U) -> Pair[U] {
    where fst = f(p.fst);
    where snd = f(p.snd);

    fst.pair(snd)
}
```

We can import that module and make use of it like so:

```rust
type pair = source["pair"];

where p = 1.pair(2); // Pair(1, 2)
where p2 = p.map(x -> x * 2); // Pair(2, 4)
```

This code is quite readable, and uses UFCS. This is the sort of code that we would like to "arrive to" as the conclusion of this document.

However, using "simple" UFCS, this is what the above code desugars to:

```rust
type pair = source["pair"];

where p = pair(1, 2);
where p2 = map(p, x -> x * 2);
```

As you can see, neither the `pair` nor the `map` function exist in the current scope - this
should lead to two interpreter errors. A "simple" solution is to force the user to "re-export"
all the functions they will be using through UFCS:

```rust
type pair = source["pair"];
where map = pair.map;
where pair = pair.pair;

where p = 1.pair(2); // Pair(1, 2)
where p2 = p.map(x -> x * 2); // Pair(2, 4)
```

But this requires one extra binding per UFCS call, and can quickly get annoying. Without even mentioning binding conflicts and shadowing:

```rust
type pair = source["pair"];
where map = pair.map;
where pair = pair.pair;

where p = 1.pair(2); // Pair(1, 2)
where p2 = p.map(x -> x * 2); // Pair(2, 4)

// now we want to use the `map` function from the `list` module:

type list = source["list"];

where l = [1, 2, 3].map(x -> x * 2);
```

The second call to `.map()` would now resolve to `pair.map([1, 2, 3], x -> x * 2)`, which would cause a confusing type error for the user - we actually want it to resolve to `list.map` in that case.
An option is to create a new `map` binding in the source:


```rust
type pair = source["pair"];
where map = pair.map;
where pair = pair.pair;

where p = 1.pair(2); // Pair(1, 2)
where p2 = p.map(x -> x * 2); // Pair(2, 4)

// now we want to use the `map` function from the `list` module:

type list = source["list"];
where map = list.map;

where l = [1, 2, 3].map(x -> x * 2);

// but we want to map pairs again
where psquared = p2.map(x -> x * x);
```

`p2.map(x -> x * x)` will now resolve to a call to `list.map(p2, x -> x * x)`, which is not what the user had in mind either! The only option in that system is to create a binding with a different name for `list.map`:

```rust
type pair = source["pair"];
where map = pair.map;
where pair = pair.pair;

where p = 1.pair(2); // Pair(1, 2)
where p2 = p.map(x -> x * 2); // Pair(2, 4)

// now we want to use the `map` function from the `list` module:

type list = source["list"];
where lmap = list.map; // lmap instead of map

where l = [1, 2, 3].lmap(x -> x * 2);

// but we want to map pairs again
where psquared = p2.map(x -> x * x);
```

Which is... not ergonomic. And confusing! If you were to read someone else's code, you would be confused if they used a different naming convention compared to yours - for exmaple, if they were binding `list.map` to `mapL` instead of `lmap`. This gets worse with more complex functions and function names.

The best solution would be for the code to look like this:

```rust
type pair = source["pair"];
type list = source["list"];
// no re-exports

where p = 1.pair(2); // Pair(1, 2)
where p2 = p.map(x -> x * 2); // Pair(2, 4)

where l = [1, 2, 3].map(x -> x * 2);

where psquared = p2.map(x -> x * x);
```

which the type system would desugar appropriately to the following:

```rust
type pair = source["pair"];
type list = source["list"];
// no re-exports

where p = pair.pair(1, 2); // Pair(1, 2)
where p2 = pair.map(p, x -> x * 2); // Pair(2, 4)

where l = list.map([1, 2, 3], x -> x * 2);

where psquared = pair.map(p2, x -> x * x);
```

## Proposed solution

1. Have more complex method resolution rules than just UFCS
2. Allow it to resolve to default members of types?
  1. How do we restrict this so that it creates no ambiguities and makes sense?
  2. What if it resolves to a binding's field instead? that must make for some pretty confusing errors

The TL;DR is as follows:

Allow UFCS to desugar `x.foo()` to `foo(x)` OR `_.foo(x)`

with `_` being any binding or type in the current namespace.

In the case of an ambiguity, emit a hard error.

## Drawbacks

1. Resolving to a binding's field instead of a module function

```rust
// int.jk
func add(lhs: int, rhs: int) -> int { lhs + rhs }

// main.jk
type int = source["int"];

type Random(add: (int, int) -> int);

func sub(lhs: int, rhs: int) -> int { lhs - rhs }

where random_binding = Random(add: sub); // !!

12.add(15);
```

If we do allow resolving to binding's fields and type fields and functions that exist in the current namespace, then this could resolve to either of the following:

- `int.add`
- `random_binding.add`, which is actually the `sub` function.

What to do in that case? What can the user do? We can definitely error on ambiguities, but surely this would incur complexity for users.

## Proposed solution 2

Skip UFCS and instead introduce a piping operator.

```rust
type pair = source["pair"];
type list = source["list"];

where p = 1 |> pair.pair(2);
where p2 = p |> pair.map(x -> x *2);

where l = [1, 2, 3] |> list.map(x -> x * 2);

where psquared = p2 |> pair.map(x -> x * x)
```

This syntax is however very heavy for what should be a small typed scripting language.