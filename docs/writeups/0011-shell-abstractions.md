# WU0011: Shell abstraction

`jinko` being a scripting language, interacting with the user's shell is a priority. We need to be able to build solid abstractions over shell commands as well as functionality such as piping, redirection, command chaining, etc, while still being able to inspect a command's output, error output, and exit code.

The idea solution would be something centered around types - but how, and which types? How do we define them? How do we *compile* them?

## Ideas

1. Extensible `Shell` type

## Extensible `Shell` type

By having a base generic `Shell` type which has a couple methods, such as `.arg()` and `.execute()`, we can abstract almost all shell command building functionality before executing said command. It probably needs a couple more methods to enable piping functionality and redirection.

```rust
// shell.jk

type Shell[T = void](
    cmd: string,
    args: Vector[string] = Vector(), // default argument to an empty vector, so we can just do `Shell(cmd: ...)`
);

type Output(exit_code: int, stdout: string, stderr: string);

func arg[T](sh: Shell[T], arg: string) -> Shell[T] {
    Shell(cmd: sh.cmd, args: sh.args.push(arg))    
}

func execute[T](sh: Shell[T]) -> Output {
    where cmd_string = sh.args.fold(sh.cmd,
        (cmd_string, arg) -> cmd_string.concat(' ').concat(arg));

    where exit_code = std.os.system(cmd_string);

    // some magic to get stdout and stderr

    Output(exit_code, stdout, stderr)
}

// specialization for "cat"
type Shell[T: "cat"](
    // missing some options but w/ever
    show_numbers: bool = false,
    show_tabs: bool = false,
    show_ends: bool = false,
    args: Vector[string] = Vector(),
);

func show_numbers(sh: Shell["cat"]) -> Shell["cat"] {
    Shell["cat"](show_numbers: true, ..sh)
}
func show_tabs(sh: Shell["cat"]) -> Shell["cat"] {
    Shell["cat"](show_tabs: true, ..sh)
}
func show_ends(sh: Shell["cat"]) -> Shell["cat"] {
    Shell["cat"](show_ends: true, ..sh)
}
```

```rust
// main.jk

type shell = source("shell");
type Shell = shell.Shell;
type Output = shell.Output;

where output = Shell["cat"]()
    .show_numbers()
    .show_tabs()
    .arg("a_specific_file.txt")
    .execute();
```
