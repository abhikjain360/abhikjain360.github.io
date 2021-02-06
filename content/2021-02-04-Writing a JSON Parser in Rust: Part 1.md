+++
title = "Writing a JSON Parser in Rust: Part 1"
date = 2021-04-02

[taxonomies]
tags = ["Rust", "JSON"]
categories = ["programming"]
+++

One of the best ways to get started with parsing strings in a particular programming language is to write a JSON parser, so that's what we will do. I don't care terribly about performance, but the code should be easy to write, and written well enough for me to understand when I re-read it again (it often happens that I code so terribly bad or don't document enough that I am not able to understand what I wrote, I am sure you must have had same experience before 😝).
In case you want to jump to code directly, the completed code can be found [here][Github Link].

> Note that if you want a really good and feature complete JSON parser, try out [serde][Serde JSON].

### Why use Rust ?

Mainly because I like Rust, and because I hope to make other people realise how awesome Rust is. Probably using Python would have been easier, but anyone who has had experience with Rust will probably tell you Rust if more elegant and fun to code!

<img
    src="https://rustacean.net/assets/cuddlyferris.png"
    style="max-width: 25%; height: auto; margin-left: auto; margin-right: auto;"
></img>


### Why use JSON ?

JSON has really easy syntax, and is used often enough that learning how to parse it might turn out to be useful. This is not to say that there aren't other simpler things to parse (like Toml, Yaml etc.), but I feel JSON is more widely used.

### Pre-requisite

I will assume some familiarity with programming in general, and knowledge about [JSON's syntax][JSON homepage]. You will need to have [Rust setup][Rustup] and some knowledge about Rust as well. If you are just starting out I highly recommend the [Rust Book][Rust Book].

### Setup

If you know how to create a new rust project, you can skip this part. We will use [cargo][Cargo] to setup our project. Simply run in a terminal:

```bash
cargo new json-parser
```

It will create a folder named `json-parser` in directory where ran the command.

```
|-- Cargo.toml
`-- src
    `-- main.rs
```

The code which executes resides in `src/main.rs`. If we look at the contents, we will find a 'Hello World' program coded in it.

```rust
fn main() {
    println!("Hello, world!");
}
```

To compile the code, simply run from root directory:
```
cargo run
```
or to run with optimizations:
```
cargo run --release
```


### Creating Modules

In any codebase, it is always nice to split up code into various files. In this mini-project we will create 3 additional files:
- `values.rs` : To define the `Value` enum, which will denote the type of JSON data.
- `parser.rs` : Will have code to parse the JSON data in `str` form,
- `macros.rs` : This will contain code in which we define macros to allow use to easily deal with JSON data.

So we create these files in the `src` directory, so the contents now look like:

```
|-- Cargo.toml
`-- src
    |-- macros.rs
    |-- main.rs
    |-- parser.rs
    `-- values.rs
```

We also import all the files in `main.rs`, so now it looks like:

```rust
// main.rs

#![allow(dead_code)]

mod macros;
mod parser;
mod values;

fn main() {
    println!("Hello, world!");
}
```

The `#![allow(dead_code)]` at the top is to turn of warning. In general Rust compiler is really aggressive with warnings about dead code, but for now they are just annoying so we disable them. Using them in the `main.rs` file is enough to disable warnings for the entire project.

### The Plan

Before we set out to write our code, we should create a rough sketch of what we want to create. We will use `std::collection::HashMap`, and data structure provided by Rust's standard library which is somewhat similar to Python dictionaries of C++ `std::unordered_map` in terms of functionality. We want the JSON data to be stored as key-value pairs in the *HashMap*, and able to generate it  by simply passing a `str` type.

Some macros to use this functionality easily would also be nice.

### `Value` enum

Rust has a very [powerful and versatile enums][miss Rust enums], and we are going to use them to denote in general any types that might come up JSON. Also, because Rust is statically typed, we can not just assign variables with random data (like in Python) or use void pointer (like you might do in C/C++). The program simply won't compile. So instead we create a enum with wraps around all the data types. We write this enum in `src/values.rs`.

```rust
// values.rs

use std::collections::HashMap;

#[derive(Debug)]
pub enum Value {
    Str(String),
    Number(i32),
    Float(f32),
    Bool(bool),
    Array(Vec<Value>),
    Object(HashMap<String, Value>),
    Null
}
```

In Rust, enum variants can store inside them another data type, and enums can also very easily be pattern-matched, which can be useful for error handling. `Vec` is just a dynamically-sized array provided by the Rust's standard library. The types also have angular brackets (`< >`) after them are using **generics**, similar to C++'s *templates*.

The `HashMap` type takes in 2 generics, one for the *key* type with which we index and one for *value* type which is the type which is returned. In our case we index with a `String`, and get a variant of `Value` enum.

At the top of the struct we derived the *Debug* trait, which auto-magically allows us to print the data type, without us needing to define or implement anything. It is also a type of macro, called [derive macros][derive macros]. So we can basically do something like:

```rust
let val = Value::Number(123);
println!("{:?}", val);
```

and this will give us the output:

```
Number(123)
```

**Traits** in Rust are similar to *interfaces* in Java of *virtual classes* in C++, though much simpler so that they are useful but don't do too much either. Traits allow us to perform *duck-typing* to some extend (though we won't use it for this purpose here).

Lastly, the `pub` keyword allows us to use this enum in other modules as well.

### `Parser` struct

In order to parse the given string we create a struct `Parser`, which will hold a *peekable* iterator to the string, and the iterator will return a character on each `next()` call. We use the standard library's `std::str::Char`, which is just the iterator we need. In Rust, anything that implements the `Iterator` trait becomes a iterator, which provides us with tons of functionalities. Another struct from standard library we use ise `std::iter::Peekable`, which is also an iterator, just allows us to *peek* the values instead of consuming them.

So our struct looks like

```rust
// src/parser.rs

use std::iter::Peekable;
use std::str::Chars;

pub struct Parser<'a> {
    src: Peekable<Chars<'a>>,
}
```

Notice that we have given `Parser` a generic `'a`, which is a *lifetime generic*. One of the things Rust guarantees is that if you are allowed to hold a reference to some data, then the data is still valid. You can hold a:

- mutable reference: meaning you have exclusive access to the data
- immutable reference: meaning that there exist multiple references to the data and none of them has exclusive access.

This allows Rust to prevent data races and other memory problems that are a [major cause of bugs in other languages][Microsoft's report]. These checks are performed at compile time to ensure that there is no performance penalty at runtime (though you can always use [Rc][Rc docs] for runtime checks). To do this, Rust introduced the concept of lifetime. Each piece of data or variable has a lifetime associated with it, which denotes for how long during the execution of the program will that piece of data be valid/exist, and compiler doesn't allow references to exist beyond the lifetime of the data. There are ways to circumvent this checking by the compiler, as the compiler isn't perfect and is not able to see how long the data will be valid in all possible cases.

So in our case, the `std::str::Chars` takes a immutable reference to the string it iterates over, and thus we need to provide the struct with a lifetime to tell the compiler that the struct `Parser` created should be allowed to live as long as the reference to the string is valid.

Now let's add some methods to our struct. We do this by creating an `impl` block:

```rust
// parser.rs

impl <'a> for Parser<'a> {
    pub fn new(src: &'a str) -> Self {
        Parser {
            src: src.chars().peekable(),
        }
    }
}
```

Using `new` is the standard way most structs are created in Rust, so we also define a similar way to construct the `Parser` struct. We take a immutable reference to the string being parsed as we don't need to modify it, only read from it. The lifetime with the reference `&'a` tells the compiler to check that `src` is valid atleast as long as the created `Parser` lives.

> Usually Rust compiler is able to figure out lifetimes using *lifetime elisions*, but in our case it is not. To know about cases when compiler does figure out lifetimes on its own, see [this][Lifetime elisions Rust Refence].

The `chars()` method on a `str` returns a `std::str::Chars`, and we make it peekable.

This is it for the 1st part! In the net part we will write some more methods for the `Parser` struct, and actually parse the string to produce the HashMap.

[Github Link]: https://github.com/abhikjain360/json-parser
[Serde JSON]: https://docs.serde.rs/serde_json
[JSON homepage]: https://www.json.org/json-en.html
[Rustup]: https://rustup.rs/
[Rust Book]: https://www.json.org/json-en.html
[Cargo]: https://doc.rust-lang.org/stable/cargo/
[Rust Macros]: https://doc.rust-lang.org/book/ch19-06-macros.html
[miss Rust enums]: https://www.reddit.com/r/rust/comments/l594zl/everywhere_i_go_i_miss_rusts_enums/
[derive macros]: https://doc.rust-lang.org/reference/procedural-macros.html#derive-macros
[Microsoft's report]: https://msrc-blog.microsoft.com/2019/07/16/a-proactive-approach-to-more-secure-code/
[Rc docs]: https://doc.rust-lang.org/stable/std/rc/struct.Rc.html
[Lifetime elisions Rust Refence]: https://doc.rust-lang.org/reference/lifetime-elision.html#lifetime-elision
