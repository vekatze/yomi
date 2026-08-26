# argot

`argot` is an option parser for the [Neut](https://vekatze.github.io/neut/) programming language.

## Installation

```sh
neut get argot https://github.com/vekatze/argot/raw/main/archive/0.2.0.tar.zst
```

## Types

### Basics

```neut
// an opaque type that holds the internal state of the parser
data argot-kit

// creates an `argot-kit`
define make-argot-kit(args: list(string)) -> argot-kit

// creates an `argot-kit` from argv
define make-argot-kit-from-argv() -> argot-kit

// the type of parsers
alias argot(a) {
  (k: &argot-kit) -> either(error, a)
}

// convert an error to human-readable text
define report(e: error) -> string
```

### Parsers

```neut
// makes given parser optional
inline optional<a>(p: argot(a)) -> argot(?a)

// runs given parser repeatedly until it fails
inline many<a>(!p: argot(a)) -> argot(list(a))

// `attempt(p)` is the same as `p` if `p` succeeds
// `attempt(p)` rewinds the input state if `p` fails
inline attempt<a>(p: argot(a)) -> argot(a)

// executes `p1`; if it fails, executes `p2`
// note that `branch(p1, p2)` does not rewind the input state automatically
inline branch<a>(p1: argot(a), p2: argot(a)) -> argot(a)

// succeeds only if there are no remaining arguments
constant end-of-input: argot(unit)
```

### Presets

```neut
// parses a flag (`--enable-color`)
inline flag(key: &string) -> argot(bool)

// parses an integer (`--my-key 1234`)
inline int-required(key: &string) -> argot(int)

// parses a float (`--my-key 23.45`)
inline float-required(key: &string) -> argot(float)

// parses a string (`--clang-option "-fsanitize=address"`)
inline string-required(key: &string) -> argot(string)

// parses a positional argument (the `foo` in `my-command foo`)
constant string-argument: argot(string)
```

### Utilities

```neut
// gets the number of remaining arguments
define get-num-of-items(k: &argot-kit) -> int

// exports the current parser state
define export-state(k: &argot-kit) -> list(string)

// replaces the current parser state with `xs`
define import-state(k: &argot-kit, xs: list(string)) -> unit

// state == ["foo", "bar", "baz", "qux"]
// ↓ pop-key(k, "bar")
// state == ["foo", "baz", "qux"], output == True
define pop-key(k: &argot-kit, key: &string) -> bool

// state == ["foo", "bar", "baz", "qux"]
// ↓ pop-key-value(k, "bar")
// state == ["foo", "qux"], output == Right("baz")
define pop-key-value(k: &argot-kit, key: &string) -> either(error, string)

// state == ["foo", "bar", "baz", "qux"]
// ↓ pop-head(k)
// state == ["bar", "baz", "qux"], output == Right("foo")
define pop-head(k: &argot-kit) -> either(error, string)
```

## Example

```neut
import {
  core::either {from-right},
  core::int.io {print-int},
  core::list {for},
  this::argot {argot},
  this::argot-kit {make-argot-kit-from-argv},
  this::error {report},
  this::parse {end-of-input, many, optional},
  this::preset {flag, int-required, string-required},
}


// defines a type that stores the result of argument parsing
data config {
| Config(v1: bool, v2: bool, v3: int, v4: string, v5: list(string))
}

// defines an argument parser
constant my-argument-parser: argot(config) {
  (k) => {
    try v1 = flag("-b")(k);
    try v2 = flag("-c")(k);
    try mv3 = optional(int-required("--integer"))(k);
    try mv4 = optional(string-required("--clang-option"))(k);
    try v5 = many(string-required("-i"))(k);
    try _ = end-of-input(k);
    Right(
      Config{
        v1,
        v2,
        v3 := from-right(mv3, 123),
        v4 := from-right(mv4, *""),
        v5,
      },
    )
  }
}

// tries the argument parser
define zen() -> unit {
  pin k = make-argot-kit-from-argv();
  match my-argument-parser(k) {
  | Left(e) =>
    pin message = report(e);
    print-line(message)
  | Right(v) =>
    let Config{v1, v2, v3, v4, v5} = v;
    let _ = v1;
    let _ = v2;
    let _ = v4;
    print("right\n");
    print("v3: ");
    print-int(v3);
    print("\nv5:\n");
    for(v5, (t) => {
      pin t = t;
      print-line(t)
    })
  }
}
```

The parser `my-argument-parser` can handle arguments like the following:

```text
-b --integer 123 -i test -i hoge
```
