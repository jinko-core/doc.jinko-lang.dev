# Online REPL landing page

Just throwing ideas out there

```rust
type Interjection = "hi";
type Name = "world" | "jinko";

func greet(start: Interjection, to_greet: Name) {
  println("{start} {name}!")
}

greet("jinko", "world");
```

```rust
error: cannot widen type `"jinko"` to value of type `"hi"`
note: hi to you too!
```
