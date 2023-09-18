# First class modules

This writeup explores the possibility of removing the `incl <source>` syntax for a more functional approach to "includes" or modules as a whole.

## Existing solution
## Problems
  1. Lack of scoping at the time
  2. Need special handling
  3. Need special handling for inner functions/types
  4. Need special path resolution - not present with regular `jinko` code

## Proposed solution

### The `Source` type

bikeshedding:

- `source`?
- `mod`?
- `module`?
- `incl`?
- `with`?
- `import`

### Issues

1. Requires first class types

Let's say that we want to source the following `utils` module:

```rust
// foo.jk

type Left[L](inner: L);
type Right[R](inner: R);
type Either[L, R] = Left[L] | Right[R];

func is_left[L, R](value: Either[L, R]) -> bool {
    switch value {
        _: Left -> true,
        _ -> false,
    }
}
```

Accessing the `is_left` function with the proposed solution is quite easy - our magic type will have an `is_left` member of type `func[L, R](Either[L, R]) -> bool`;
But how do we actually access the `Either` type? `utils.Either` would be nice, but it does mean that the generate `utils` type needs a field named `Either` which would be of type... type?
  
#### First solution: Type specialization crimes

1. Add type specialization
2. Generate a basic `util[T = ()]`  type which contains all of the members of our module *except* for the types.
3. Specialize `util` as each type within the module, so something like this

```rust
type utils = source["utils"];

// becomes

type utils[T = ()](
    is_left: func[L, R](Either[L, R]) -> bool,
)

type utils[L, T: Left](inner: L);
type utils[R, T: Right](inner: R);
type utils[L, R, T: Either] = utils[Left][L] | utils[Right][R];
```

which we can use like so:

```rust
// main.jk

where l = utils[Left](inner: 156);
where value: utils[Either] = l;

utils.is_left(value);
```

* Problems with this solution
    * Different syntax from module functions and module variables
    * Unwieldy
    * Ugly as sin
    * Requires type specialization - do we want that?

#### Second solution: First class types

```rust
type utils = source["utils"];

// becomes

type utils(
    Left: type,
    Right: type,
    Either: type,
    is_left: func[L, R](utils.Either[L, R]) -> bool,
)
```

which we can use like so:

```rust
// main.jk

where l = utils.Left(inner: 156);
where value: utils.Either = l;

utils.is_left(value);
```

* Problems with this solution
    * Requires self referential types (`utils.is_left` is of type `func[L, R](utils.Either[L, R]) -> bool`)
    * Requires first class type support
