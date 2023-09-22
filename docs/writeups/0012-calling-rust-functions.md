# WU0012: Calling Rust functions

## Syntax experimentation

```rust
fn foo() -> i32 { 15 }
fn bar(a: i32, c: i32) -> i32 { a + c }
```

```rust
where rust = source("rust");
where i32 = rust.i32;
where Fn = rust.Fn;

where foo_type = rust.fn("foo", return_type: i32).build();
where bar_type = rust.fn("bar", args: (i32, i32), return_type: i32).build();

where foo_type = Fn[(), i32](name: "foo");
where bar_type = Fn[(i32, i32), i32](name: "bar");
```
