# WU0012: Default field accesses

## Proposal

Introduce a new operator `::` to access a type's default field value

## Reasoning

The `source` function exposed in WU0009 is more likely to return a new type than an instance of that type, for numerous reasons including module member overriding and simplicity. However, one question remains: how to access the members of a module without first creating an instance of that type?

```rust
type List = source("list");

where l = List.map([1, 2, 3], x -> x * 2);
```

In the above example, `List` is not an instance of the type `List` - it is the actual List *type*. You can imagine that it contains the following fields:

```rust
type List(
  map: func[T, U](Array[T], func(T) -> U) -> Array[U] = /* default map function */
);
```

Hence, accessing the `map` field requires an instance of type `List` - so we must first create one:

```rust
type List = source("list");
where list = List;

where l = list.map([1, 2, 3], x -> x * 2);
```

This is unwieldy, boilerplate-y and not very fun to work with. The syntax from the first code snippet makes a lot more sense, and is a lot more manageable. However, it does not make *sense* to allow accessing a type's field without an instance of that type, and it could be quite error prone.

## Explanation

Adding an `::` operator to *explicitly* indicate that we would like to use a type's field's default value (provided it exists) makes intent clearer, while still keeping syntax simple. It also has the advantage of still allowing module member overriding.

## Examples

```rust
type List = source("list");

where l = List::map([1, 2, 3], x -> x * 2);
```

```rust
type Foo = source("foo");
// has module `Bar`, which has module `Baz`, which has type `Qux`

where q = Foo::Bar::Baz::Qux(x: 15, y: 14);
```

or

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
