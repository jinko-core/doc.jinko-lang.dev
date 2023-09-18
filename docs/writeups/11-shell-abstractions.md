# Shell abstraction

`jinko` being a scripting language, interacting with the user's shell is a priority. We need to be able to build solid abstractions over shell commands as well as functionality such as piping, redirection, command chaining, etc, while still being able to inspect a command's output, error output, and exit code.

The idea solution would be something centered around types - but how, and which types? How do we define them? How do we *compile* them?
