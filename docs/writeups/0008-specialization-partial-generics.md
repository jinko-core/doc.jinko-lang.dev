# WU0008: Specialization, generic application, partial generic application

Let's consider `jinko`'s memory release mechanism. If the goal is to make copies explicit and moves implicit, one can explicitly ask for a value to be released (and for its memory to be released by the garbage collector)
using a simple moving function such as this one;

```rust
type MegaBigDynamicStruct(a: Vector[Vector[string]] /* wahou! */)

func consume(value: MegaBigDynamicStruct) {}

where mbds = MegaBigDynamicStruct(a: Vector::new().append(Vector::new().append("so dynamic!"))));

mbds.consume();

// error: use of moved value mbds
where first_string = mbds.a.at(0).at(0);
```

We can generalize this mechanism and offer a generic `consume` function. Since the garbage collector is not guaranteed to run immediately, it's more like you're marking a value for release rather than releasing it. Let's call
this function `rot`, as you're indicating that you're leaving this value to decay and won't be using it anymore.

```rust
func rot[T](value: T) {}
```

...and that's it! We now have a generic function which takes care of indicating to the GC that this value's memory can be reclaimed. We can imagine something like this for `jinko`:

```rust
func foo() -> Baz {
    where a = bar();
    where b = baz();

    print("{a}");

    b
}
```

One pass the interpreter should have should be to insert calls to `rot` when possible, in order to reclaim memory. Because in this function `foo`, `a` goes out of scope at the end and is not used or returned afterwards like `b`,
we can imagine the following code after said "rot-pass", after the last uses of `a`.

```rust

func foo() -> Baz {
    where a = bar();
    where b = baz();

    print("{a}");

    a.rot();
    b
}
```

Now, this is all good and dandy, but we can extend this `rot` mechanism to incur interesting behavior on memory release. For example, a type which takes care of managing an operating system resource, like a file descriptor or
dynamic memory.

```rust
ext func open(path: string, flags: int);
ext func read(path: string, flags: int);

type Error(cause: string);
type File(path: string, fd: Maybe[int] = Nothing);

func open(f: File) -> Result[File, Error] {
    switch f.fd {
        fd: int => Error(cause: "file {f.path} is already opened"),
        _: Nothing => {
            where new_fd = open(f.path, 0);
            if new_fd == -1 {
                Error(cause: "couldn't open file")
            } else {
                File(path: fd.path, fd: new_fd)
            }
        }
    }
}

func read(f: File) -> Result[string, Error] {
    switch f.fd {
        fd: int => {
            where mut buf = "";
            where ret_val = read(fd, buf.ref_mut(), 1)

            buf
        }
        _: Nothing => f.open().try().read(),
    }
}

func foo() {
    where test_file = File("test.jk");

    // here, the interpreter will insert a call to test_file.rot()
}
```

when `test_file` goes out of scope here, it would be nice for the underlying file descriptor to get closed. This way, we can return the resource to the OS and avoid leaking resources. One way to do so would be to *specialize* the behavior of `rot` for `File` instances,
so that it performs a call to `close`. Let's explore the various syntaxes to achieve this.

```rust
ext func close(fd: int);

func close(f: File) {
    switch f.fd {
        fd: int => close(fd),
        _ => {}
    }
}

// specialize the rot function for `File`
func rot[???](f: File) {
    f.close()
}
```

How would the syntax for specifying the generic type `T` to be `File` look like?

One proposal is as follows
```rust
func rot[T = File](f: File) {
    f.close()
}
```

This syntax is nice, but does not really allow partial specialization. We could want something like this `rot` function for vectors or tuples
```rust
func rot[T = Vector[T]](v: Vector[T]) {
    for value in v {
        value.rot()
    }  
}
```


Having an implicit rule for reusing the same generics does not work really well and could get confusing
```rust
// what is the difference between
func rot[T = Vector[T]](v: Vector[T]);
// and
func rot[T = Vector[int]](v: Vector[int]);
// and
func rot[T = Vector[undeclared_type]](v: Vector[undeclared_type]);
// which one should error out?
```

Where is the type `T` used within `Vector[T]` defined? Is that a type error because `Vector[T]` refers to a type `T` which is undefined? Should the syntax be something like this to explicitly declare the second generic?
```rust
func rot[U][T = Vector[U]](v: Vector[U]); // ?
func rot[T = Vector[U]](v: Vector[U])[U]; // ?
```

This is basically a partial application of generics. If we solve that syntax issue, we can probably reuse the same syntax for partial function application.

The philosophy of generics is that they are usually declared before being used, for example in the case of the generic `rot` or `id`.

```rust
// declared here...
//       |
//       |   and used here
//       |        |     |
//       v        v     v
func id[T](value: T) -> T {
    value
}
```

For application however, the specialized type comes after the function or type being used:

```rust
type Generic[T](value: T);

//      generic applied here
//                vvv
where g = Generic[int](15);
```

so one could think of specialization as generic application, but on a function declaration instead of type
```rust
func rot[T](value: T)[File] {
    // value is a `File`
}
```

but what about the following examples?
```rust
// is that an error?
func rot[T](value: File)[File] {}

// where is `T` from `Vec[T]` defined?
func rot[T](value: T)[Vector[T]] {}
// is this better? where is `U` defined?
func rot[T](value: T)[Vector[U]] {}
// what's the difference with this?
func rot[T](value: T)[Vector[undeclared_type]] {}
func rot[T](value: T)[Vector[int]] {}
```

Or we could have entirely different partial function specialization syntax, which could mean "returns a function specializable on a generic `U`"

```rust
func rot[T = Vec[U]](value: Vec[U]): [U] {
    
}

// gets a bit uglier here...
func id[T = Vector[U]](value: Vector[U]): [U] -> Vector[U] {
    value
}
```

or even forego the concept entirely, define each function as a generic type (a callable one), and specialize by defining new bindings

```rust
func rot[T](value: T) {}

where rot[f: File] = {
    f.close()
}
```

or remove the generics entirely, keeping them only in the case of partial specialization?

```rust
func rot[T](value: T) {}
func rot(value: File) { value.close() }
func rot[T](values: Vector[T]) { for value in values { value.rot() } }
```
and have error about "useless" specialization? As in, show as duplicate functions?
```rust
// function already declared here - might be a useless case of specialization?
func rot[U](value: U) {}
// consider restricting the type `U` further for partial specialization
// func rot[U](value: ...[U]) {}
```

which is probably the better solution :)

Let's take a look at a few examples where partial generic application and specialization become useful:

```rust
// declaration without a body - there's no "generic" way to format something and it should be an error
func format[T](value: T) -> string;

// specialization for the primitive types
func format(value: int) -> string {
    format_int(value)
}

func format(value: bool) -> string {
    switch value {
    true => "true",
    false => "false",
    }
}

func format(value: char) -> string {
    "".concat(value)
}

func format(value: string) -> string {
    value
}
```

The one thing sticking out to me is that the specialized versions of `format` look just like regular functions. There is no indication (syntactically) that they are specialized functions. So for example why would this
error out

```rust
func not_spec(a: int) -> int { a }
func not_spec(a: string) -> char { a.at(0) }
```

but not
```rust
func spec[T, U](a: T) -> U;
func spec(a: int) -> int { a }
func spec(a: string) -> char { a.at(0) }
```

This creates confusing errors for users and isn't a very good show of the language. We could make this good only by having extensive error messages with extremely good examples.

Or should we introduce a new function keyword to separate the original function and the specialization?

```rust
func format[T](value: T) -> string;
spec format(s: string) -> string { value }

func map[T, U](value: T, f: func(T) -> U) -> U {
    f(value)
}
spec map[T, U](value: Maybe[T], f: func(T) -> U) -> Maybe[U] {
    switch value {
        contained: T => f(contained)
        Nothing => Nothing
    }
}
spec map[T, U](values: Vector[T], f: func(T) -> U) -> Vector[U] {
    values.fold(Vector::new(), |vec, value| vec.append(value.map(f)))
}

func fold[In, Out, Elt](
    values: In,
    init: Out,
    folder: func(Out, Elt) -> Out)
-> Out {
    where mut output = init;
    for elt in values {
        output = folder(output, elt)
    }

    output
}
```

How to prevent overzealous specialization? For example, specializing `map` on `int` and `string` would not be great. Could it affect users of a library?
