---
layout: post
title: "Syntex: Syntax Extensions for Rust 1.0"
date: 2015-02-09 19:37:32 -0800
comments: true
categories: [rust, syntax extensions, syntex]
---

I use and love syntax extensions, and I'm planning on using them to simplify
down how one interacts with a system like
[serde](https://github.com/erickt/rust-serde). Unfortunately though, to write
them you need to use Rust's `libsyntax`, which is not going to be exposed in
Rust 1.0 because we're not ready to stablize it's API. That would really hamper
future development of the compiler.

It would be so nice though, writing this for every type we want to serialize:

```rust
#[derive_deserialize]
struct Point {
    x: int,
    y: int,
}
```

instead of:

```rust
impl <D: Deserializer<E>, E> Deserialize<D, E> for Point {
    fn deserialize_token(state: &mut D, token: Token) -> Result<Point, E> {
        try!(state.expect_struct_start(token, "Point"));

        let mut x = None;
        let mut y = None;

        loop {
            let idx = match try!(state.expect_struct_field_or_end(&["x", "y"])) {
                Some(idx) => idx,
                None => { break ; }
            };

            match idx {
                Some(0us) => { x = Some(try!(state.expect_struct_value())); }
                Some(1us) => { y = Some(try!(state.expect_struct_value())); }
                None | Some(_) => { panic!() }
            }
        }

        let x = match x {
            Some(x) => x,
            None => try!(state.missing_field("x")),
        };

        let y = match y {
            Some(y) => y,
            None => try!(state.missing_field("y")),
        };

        Ok(Point{x: x, y: y,})
    }
}
```

So I want to announce my plan on how to deal with this (and also publically
announce that everyone can blame me if this turns out to hurt Rust's future
development). I've started [syntex](https://github.com/erickt/rust-syntex), a
library that enables code generation using an unofficial tracking fork of
`libsyntax`. In order to deal with the fact that `libsyntax` might make
breaking changes between minor Rust releases, we will just release a minor or
major release of `syntex`, depending on if there were any breaking changes in
`libsyntax`.  Ideally `syntex` will allow the community to experiment with
different code generation approaches to see what would be worth merging
upstream. Or even better, what hooks are needed in the compiler so it doesn't
have to think about syntax extensions at all.

I've got the basic version working right now. Here's a simple
`hello_world_macros` syntax extension (you can see the actual code
[here](https://github.com/erickt/rust-syntex/tree/master/hello_world)). 
First, the `hello_world_macros/Cargo.toml`:

```toml
[package]
name = "hello_world_macros"
version = "0.2.0"
authors = [ "erick.tryzelaar@gmail.com" ]

[dependencies]
syntex = "*"
syntex_syntax = "*"
```

The `syntex_syntax` is the crate for my fork of `libsyntax`, and `syntex`
provides some helper functions to ease registering syntax extensions.

Then the `src/lib.rs`, which declares a macro `hello_world` that just produces a
`"hello world"` string:

```rust
extern crate syntex;
extern crate syntex_syntax;

use syntex::Registry;

use syntex_syntax::ast::TokenTree;
use syntex_syntax::codemap::Span;
use syntex_syntax::ext::base::{ExtCtxt, MacExpr, MacResult, TTMacroExpander};
use syntex_syntax::ext::build::AstBuilder;
use syntex_syntax::parse::token::InternedString;

fn expand_hello_world<'cx>(
    cx: &'cx mut ExtCtxt,
    sp: Span,
    tts: &[TokenTree]
) -> Box<MacResult + 'cx> {
    let expr = cx.expr_str(sp, InternedString::new("hello world"));

    MacExpr::new(expr)
}

pub fn register(registry: &mut Registry) {
    registry.register_fn("hello_world", expand_hello_world);
}
```

Now to use it. This is a little more complicated because we have to do code
generation, but Cargo helps with that. Our strategy is use a `build.rs` script
to do code generation in a `main.rss` file, and then use the `include!()` macro
to include it into our dummy `main.rs` file. Here's the `Cargo.toml` we need:

```
[package]
name = "hello_world"
version = "0.2.0"
authors = [ "erick.tryzelaar@gmail.com" ]
build = "build.rs"

[build-dependencies]
syntex = "*"
syntex_syntax = "*"

[build-dependencies.hello_world_macros]
path = "hello_world_macros"
```

Here's the `build.rs`, which actually performs the code generation:

```rust
extern crate syntex;
extern crate hello_world_macros;

use std::os;

fn main() {
    let mut registry = syntex::Registry::new();
    hello_world_macros::register(&mut registry);

    registry.expand(
        "hello_world",
        &Path::new("src/main.rss"),
        &Path::new(os::getenv("OUT_DIR").unwrap()).join("main.rs"));
}

```

Our `main.rs` driver script:

```rust
// Include the real main
include!(concat!(env!("OUT_DIR"), "/main.rs"));
```

And finally the `main.rss`:

```rust
fn main() {
    let s = hello_world!();
    println!("{}", s);
}
```

One limitiation you can see above is that we unfortunately can't compose our
macros with the Rust macros is that `syntex` currently has no awareness of the
Rust macros, and since macros are parsed outside-in, we have to leave the
tokens inside a macro like `println!()` untouched.

---

That's `syntex`. There is a bunch of more work left to be done in `syntex`
to make it really useable. There's also a lot of work in Rust and Cargo that
would help it be really effective:

* We need a way to inform Rust that this block of code is actually coming from
  a different file than the one it's processing. This is roughly equivalent to
	the [#line](https://gcc.gnu.org/onlinedocs/cpp/Line-Control.html) macros in
  `C`.
* We could upstream the "ignore unknown macros" patch to minimize
  changes to `libsyntax`.
* It would be nice if we could allow `#[path]` to reference an environment
  variable. This would be a little cleaner than using `include!(...)`.
* We need a way to extract macros from a crate.

On Cargo's side:

* It would be nice if Cargo could be told to use a generated file as the
  `main.rs`/`lib.rs`/etc.
* `Cargo.toml` could grow a plugin mechanism to remove the need to write a
  `build.rs` script.

I'm sure there's plenty more that needs to get done! So, please help out!

edit: comments on [reddit](https://www.reddit.com/r/rust/comments/2vdzd1/syntex_syntax_extensions_for_rust_10/)
