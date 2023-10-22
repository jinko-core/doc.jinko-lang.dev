# WU0012: Default field accesses

## Reasoning

The `source` function exposed in WU0009 is more likely to return a new type than an instance of that type, for numerous reasons including module member overriding and simplicity. However, one question remains: how to access the members of a module without first creating an instance of that type?

## Proposal 1

Introduce a new operator `::` to access a type's default field value

```rust
type list = source("list");

where l = list.map([1, 2, 3], x -> x * 2);
```

In the above example, `List` is not an instance of the type `List` - it is the actual List *type*. You can imagine that it contains the following fields:

```rust
type list(
    map: func[T, U](Array[T], func(T) -> U) -> Array[U] = /* default map */
);
```

Hence, accessing the `map` field requires an instance of type `List` - so we must first create one:

```rust
type list = source("list");
where list = list;

where l = list.map([1, 2, 3], x -> x * 2);
```

This is unwieldy, boilerplate-y and not very fun to work with. The syntax from the first code snippet makes a lot more sense, and is a lot more manageable. However, it does not make *sense* to allow accessing a type's field without an instance of that type, and it could be quite error prone.

## Explanation

Adding an `::` operator to *explicitly* indicate that we would like to use a type's field's default value (provided it exists) makes intent clearer, while still keeping syntax simple. It also has the advantage of still allowing module member overriding.

## Examples

```rust
type list = source("list");

where l = list::map([1, 2, 3], x -> x * 2);
```

```rust
type foo = source("foo");
// has module `bar`, which has module `baz`, which has type `Qux`

where q = foo::bar::baz::Qux(x: 15, y: 14);
```

```rust
func foo_pair(p: std::pair::Pair[int, string]) -> int { p.fst }
```

## Issues

1. How to deal with module overriding?

```rust
type list = source("list");
where list_custom = List(
    map: our_map,
);

list::map([1, 2, 3], x -> x * 2)

// these call `our_map`
list_custom::map([1, 2, 3], x -> x * 2) // works?
list_custom.map([1, 2, 3], x -> x * 2) // works for sure, but not consistent?

func our_map[T, U](l: Array[T], f: func(T) -> U) -> Array[U] { /* ... */ }
```

-> should the operator be a `field access or default access operator`?

2. module override instances vs default type accesses
3. syntax complexity
4. "method" resolution
5. Multiline chaining allowed?
6. Prefered syntax for modules - lowercaps module names?
7. How does it interact with types in general?
8. Other operators?

```rust
type Pair = std::pair::Pair;
type Pair = std\pair\Pair;
type Pair = std@pair@Pair;
type Pair = std->pair->Pair;
type Pair = std>pair>Pair;
type Pair = std~pair~Pair;
type Pair = std^pair^Pair;
type Pair = std!pair!Pair;
type Pair = std?pair?Pair;
```

## Proposal 2

Create a binding of an instance of the generated type in the enclosing scope of the receiver expression calling `source`

```rust
type list = source("list");

// becomes

type list(
    map: func[T, U](Array[T], func(T) -> U) -> Array[U] = /* default map */)
);

// binding with default initialization for all fields of the type `list`
where list = list();
```

```rust
func source(path: string) -> type {
    type new_type = source_inner(path);
    where receiver = jinko.magic.receiver_expr();

    receiver.scope().insert(
        jinko.magic.binding(receiver.name(), new_type())
    );

    new_type
}
```

## Proposal 3

Make the `.` operator access the default value if it is on a type, the field if it is on an instance

```rust
// list.jk
type Array[T] = /* ... */;
func map[T, U](arr: Array[T], f: T -> U) -> Array[U] {
    /* ... */
}

type list = source("list");
list.map([1, 2, 3], x -> x * 2);

// calls <list module>.map(Array[1, 2, 3], x -> x * 2)

where list = list.from([1, 2, 3]); // -> <list module>.Array
list.map(x -> x * 2); // calls into <list module>.map(list, x -> x * 2)
```

## Issues

1. Very magic
2. Not great for compilation
