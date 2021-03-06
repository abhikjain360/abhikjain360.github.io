+++
title = "Writing a JSON Parser in Rust: Part 2"
date = 2021-05-02

[taxonomies]
tags = ["Rust", "JSON"]
categories = ["programming"]
+++

In this blog I will write methods to parse JSON, in continuation of the [previous blog][Part 1]. So far, our `parser.rs` looks like

```rust
// src/parser.rs

use std::iter::Peekable;
use std::str::Chars;

pub struct Parser<'a> {
    src: Peekable<Chars<'a>>,
}

impl <'a> Parser<'a> {
    pub fn new(src: &'a str) -> Self {
        Parser {
            src: src.chars().peekable(),
        }
    }
}
```

But before we write our parser, let us lay the groundwork of error handling. It is not guaranteed the string passed on the the parser to parse is valid JSON. Instead of simply stopping the program we should give the user of our code some options (pun intended, for those who know Rust) to handle the errors. For this we create again create a enum, whose variants will tell the user which type of error was encountered.

```rust
// src/parser.rs

#[derive(Debug)]
pub enum ParseError {
    UnexpectedEOF,
    UnexpectedChar(char),
    ExpectedChar(char),
}
```

The syntax enum shouldn't surprise you any more, because we haven't introduced anything new since [when we defined Values enum][Part 1 Values enum]. The errors mentioned here are the only ones I am going to use, but if you wish to be more detailed about the errors, feel free to add your own variants and change the following code snippets accordingly.

# `next` and `peek`

We can call `next` on and iterator to consume the next item, and `peek` to just get a reference to the next item. But these functions don't directly return the item/reference, but instead wrap the return item in `Option<T>`, which is an enum defined in standard library as:

```rust
enum Option<T> {
    Some(T),
    None
}
```

This allows us (in most cases, *forces* us) to handle errors explicitly. There are ways to easily unwrap the function, using `unwrap` of the `?` operator. So, if we define

```rust
use crate::parser::Parser;
let parser = Parser::new(r#"{ "name": "Mr. Json" }"#);
```

then we can access the characters with
```rust
assert_eq!(parser.src.next().unwrap(), '{');
assert_eq!(*parser.src.peek().unwrap(), ' ');
```

Note that we had to dereference the pointer when using `peek()` as it returns a reference to `char`, and we Rust won't let us compare a `&char` to a `char` (which is unsound anyway for most cases). But typing the entire thing everytime we need to access the next character. So we write ourselves 2 helper functions.

```rust
// parser.rs

impl<'a> Parser<'a> {
    fn peek(&mut self) -> Result<&char, ParseError> {
        self.src.peek().ok_or(ParseError::UnexpectedEOF)
    }

    fn next(&mut self) -> Result<char, ParseError> {
        self.src.next().ok_or(ParseError::UnexpectedEOF)
    }
}
```

The `Result<T, E>` type is also defined in standard library for error handling. It is similar to `Option<T>`, except that the error also contains some information, thus there are 2 generics with it.

```rust
enum Result<T, E> {
    Ok(T),
    Err(E)
}
```

I use `Result` when I want to return some error information as well, otherwise I use `Option`. But you can use anything you like, though the convention is to use them according to their name, that is `Result` where you return a "result".

### Writing the parsers, finally

That was a lot of groundwork! But rightly so, because now we can write the parsing methods and not worry about anything else. Note that when actually coding myself I went a lot back and forth in between (I started straight with parser methods, and suffered for it), and the 2 convenience functions we wrote above weren't there in the original code. So don't think I am super-smart to have coded things in such perfect order. I just spent a lot of time on it so you don't have to!

We will divide the parsing into 3 functions:
- `skip_whitespaces()` will be used, well, skip whitespaces,
- `parse()` will be the function that will be called by user (and thus will be public), and will also deal with anything wrapped in `{}` and letting the `parse_data` figure out how to convert things to ke-value pair returing a HaspMap,
- `parse_data()` will deal with the key-value stuff, identify the datatype and returns a tupple of a *value* and it's *key*

Let's start with the easiest one, `skip_whitespaces`.


```rust
// parser.rs

use crate::values::Value;

impl<'a> Parser<'a> {
    fn skip_whitespaces(&mut self) {
        // peeking at the next char
        while let Ok(ch) = self.peek() {
            // if not whitespace, dont consume it
            if !ch.is_whitespace() {
                break;
            }
            // else consume it and move on
            self.next().unwrap();
        }
    }
}
```

We simply consume the iterator till we find a character which is not whitespace, and return without consuming the non-whitespace character. Notice the `while let` pattern matching. The while loop will run till `self.peek()` does not return an error. We could have handled the case where it returns an error (possibly by returning a `Result`), but I am not bothering right now. Feel free to do so if you wish.

Now the following methods are left.

```rust
// parser.rs

use std::collections::HashMap;

impl <'a> Parser<'a> {
    fn parse_value(&mut self) -> Result<Value, ParseError> { /*TODO*/ }
    fn parse(&mut self) -> Result<HashMap<String, Value>, ParseError> { /*TODO*/ }
}
```

The `parse_value` will return the data that comes after the key, for example in the following JSON:

```json
{
    "name": "Mr. Json",
    "age": 19
}
```

`parse_value` will deal with `"Mr. Json"` and `19`, while rest will be dealt by `parse` itself.

Let's first write the `parse()` function.

```rust
// parser.rs

impl <'a> Parser<'a> {
    fn parse(&mut self) -> Result<HashMap<String, Value>, ParseError> {
        let map = HashMap::new();

        self.skip_whitespaces();
        if self.next()? != '{' {
            return Err(ParseError::ExpectedChar('{'));
        }

        // add some code here later

        Ok(map)
    }

}
```

We are creating a new empty HashMap, and returning it. JSON always has to start with `'{'` so we are checking whether that is the case or not. If the rule is violated, we simply return the error.

Recall that `self.next()` returns a `Result<Value, ParseError>`, so we use the `?` operator on it. This operator unwraps the value if there are no errors, else it simply transmits the error through the function. We can use `?` operator only when the function in which it is being called returns the same error type as the error type of the result we are unwrapping. We can also ue the operator to unwrap `Option<T>`, but then we also need to return an `Option`.

We will be using similar pattern throughout this function. I am simply pasting the function here are code is pretty simply once you understand the previous code block. We iterate through the string until we reach `'}'`, or we encounter an error.

```rust
// parser.rs

impl <'a> Parser<'a> {
    fn parse(&mut self) -> Result<HashMap<String, Value>, ParseError> {
        let map = HashMap::new();

        self.skip_whitespaces();

        if self.next()? != '{' {
            return Err(ParseError::ExpectedChar('{'));
        }

        loop {
            self.skip_whitespaces();

            if self.next()? != '"' {
                return Err(ParseError::ExpectedChar('"'));
            }

            // getting the key
            let ch = self.next()?;
            if !ch.is_ascii_alphabetic() && ch != '_' {
                return Err(ParseError::UnexpectedChar(ch));
            }
            let mut ident = String::from(ch);
            while let Ok(ch) = self.peek() {
                if ch.is_ascii_alphanumeric() || *ch == '_' {
                    ident += &ch.to_string();
                    self.next().unwrap();
                } else {
                    break;
                }
            }

            if self.next()? != '"' {
                return Err(ParseError::ExpectedChar('"'));
            }

            self.skip_whitespaces();

            if self.next()? != ':' {
                return Err(ParseError::ExpectedChar(':'));
            }

            // getting the value and adding key-value pair to the HashMap
            map.insert(ident, self.parse_value()?);

            self.skip_whitespaces();

            // Allow single comma even after the last entry
            if *self.peek()? == ',' {
                self.next().unwrap();
            }

            self.skip_whitespaces();

            // break if closing parenthesis encountered
            if *self.peek()? == '}' {
                break;
            }
        }

        Ok(map)
    }
}
```

Next we write the `parse_value()` method. Notice that in the `parse()` method we get call `parse_value` just after we find `':'`. Thus we only need to parse the value and not the identifier, and just return the value. But before that we skip any whitespaces that might be there.

We use Rust's `match` statements, which allow us to match against patterns really easily.

```rust
//parser.rs

impl <'a> Parser<'a> {
    fn parse_value(&mut self) -> Result<Value, ParseError> {
        self.skip_whitespaces();

        let ch = *self.peek()?;
        match ch {
            // match against all possible characters
        }
    }
}
```

Rust doesn't let us get away with matchin against only certain subset of possible cases in `match`. We need to handle all possible cases. Another thing to keep in mind is that all the arms of `match` have to return the same datatype, or use the `return` keyword. Though this won't be a problem for us.

I use the following matching logic:

- `'{'` means that it's a object, and thus we simply call `parse()` on it.
- `'['` means that it's a start of an array. We call a loop and call `parse_value()` on each comma seperated item untill we reach `']'`.
- `'"'` means it a string. Again we store all the characters until another `'"'` it reached.
- `true`, `false`, or `null` are dealt with seperately.
- `'0'` to `'9'` means it's a number of a float. If it contains a decimal dot, then will be considered float alse a number.
- If none matched then return `Err(UnexpectedChar)`.

And thus we complete the function.

```rust
//parser.rs

impl <'a> Parser<'a> {
    fn parse_value(&mut self) -> Result<Value, ParseError> {
        self.skip_whitespaces();

        let ch = *self.peek()?;
        match ch {
            '{' => self.parse().map(|map| Value::Object(map)),
            '[' => {
                self.next().unwrap();
                self.skip_whitespaces();

                let mut v = Vec::new();

                while let Ok(ch) = self.peek() {
                    if *ch == ']' {
                        self.next().unwrap();
                        break;
                    } else {
                        v.push(self.parse_value()?);
                    }

                    self.skip_whitespaces();
                    if let Ok(ch) = self.peek() {
                        if *ch == ',' {
                            self.next().unwrap();
                        }
                    }
                }

                Ok(Value::Array(v))
            }
            '"' => {
                self.next().unwrap();
                let mut s = String::new();

                while let Ok(ch) = self.next() {
                    if ch == '"' {
                        break;
                    }
                    s += &ch.to_string();
                }

                Ok(Value::Str(s))
            }
            't' | 'f' | 'n' => {
                let mut s = String::from(self.next().unwrap());

                while let Ok(ch) = self.next() {
                    if !ch.is_ascii_alphabetic() {
                        break;
                    }
                    s += &ch.to_string();
                }

                match s.as_str() {
                    "true" => Ok(Value::Bool(true)),
                    "false" => Ok(Value::Bool(false)),
                    "null" => Ok(Value::Null),
                    _ => Err(ParseError::UnexpectedChar(ch)),
                }
            }
            '0'..='9' => {
                let mut s = String::from(self.next().unwrap());
                let mut is_float = false;

                while let Ok(ch) = self.peek() {
                    if *ch == '.' {
                        is_float = true;
                        s += &ch.to_string();
                    } else if ch.is_numeric() {
                        s += &ch.to_string();
                    } else {
                        break;
                    }
                    self.next().unwrap();
                }

                if is_float {
                    Ok(Value::Float(s.parse().unwrap()))
                } else {
                    Ok(Value::Number(s.parse().unwrap()))
                }
            }
            _ => Err(ParseError::UnexpectedChar(self.next().unwrap())),
        }
    }
}
```


[Part 1]: https://abhikjain360.github.io/writing-a-json-parser-in-rust-part-1/
[Part 1 Values enum]: https://abhikjain360.github.io/writing-a-json-parser-in-rust-part-1/#value-enum
