---
layout: post
title: "Serde 0.4.0 - Syntax Extensions in Stable Rust and More!"
date: 2015-05-18 07:57:19 -0700
comments: true
categories: [rust, serialization, serde, aster, syntex]
---

Hello Internet! I'm pleased to announce
[serde](https://github.com/serde-rs/serde) 0.4.0, which now supports many new
features with help from our growing serde community. The largest is now serde
supports syntax extensions in stable Rust by way of
[syntex](https://github.com/erickt/rust-syntex). syntex is a fork of Rust's
parser library libsyntax that has been modified to enable code generation.
serde uses it along with a
[Cargo build script](http://doc.crates.io/build-script.html) to expand the
`#[derive(Serialize, Deserialize)]` decorator annotations. Here's how to use
it.

First, lets start with a simple serde 0.3.x project that's forced to use
nightly because it uses `serde_macros`. The `Cargo.toml` is:

```toml
[package]
name = "hello_world"
versio = "0.1.0"
authors = ["Erick Tryzelaar <erick.tryzelaar@gmail.com>]
license = "MIT/Apache-2.0"

[dependencies]
serde = "*"
serde_macros = "*"
```

And the actual library is `src/lib.rs`:

```rust
#[feature(custom_derive, plugin)]
#[plugin(serde_macros)]

extern crate serde;

#[derive(Serialize, Deserialize)]
pub struct Point {
    x: u32,
    y: u32,
}
```

In order to use Stable Rust, we can use the new `serde_codegen`. Our strategy
is to split our input into two files. The first is the entry point Cargo will
use to compile the library, `src/lib.rs`. The second is a template that
contains the macros, `src/lib.rs.in`. It will be expanded into
`$OUT_DIR/lib.rs`, which is included in `src/lib.rs`. So `src/lib.rs` looks
like:

```rust
extern crate serde;

include!(concat!(env!("OUT_DIR"), "/lib.rs"));
```

`src/lib.rs.in` then just looks like:

```rust
#[derive(Serialize, Deserialize)]
pub struct Point {
    x: u32,
    y: u32,
}
```

In order to generate the `$OUT_DIR/lib.rs`, we'll use a Cargo build script.
We'll configure `Cargo.toml` with:

```toml
[package]
name = "hello_world"
versio = "0.1.0"
authors = ["Erick Tryzelaar <erick.tryzelaar@gmail.com>]
license = "MIT/Apache-2.0"
build = "build.rs"

[build-dependencies]
syntex = "*"
serde_codegen = "*"

[dependencies]
serde = "*"
```

Finally, the `build.rs` script itself uses `syntex` to expand the syntax
extensions:

```rust
extern crate syntex;
extern crate serde_codegen;

use std::env;
use std::path::Path;

fn main() {
    let out_dir = env::var_os("OUT_DIR").unwrap();

    let src = Path::new("src/lib.rs.in");
    let dst = Path::new(&out_dir).join("lib.rs");

    let mut registry = syntex::Registry::new();

    serde_codegen::register(&mut registry);
    registry.expand("", &src, &dst).unwrap();
}
```

Downside 1: Error Locations
---------------------------

While `syntex` is quite powerful, there are a few major downsides. Rust does
not yet support the ability for a generated file to provide error location
information from a template file. This means that tracking down errors requires
manually looking at the generated code and trying to identify where the error
in the template. However, there is a workaround.  It's actually not that
difficult to support `syntex` and the Rust Nightly compiler plugins. To update
our example, we'll change the `Cargo.toml` to:

```toml
[package]
name = "hello_world"
versio = "0.1.0"
authors = ["Erick Tryzelaar <erick.tryzelaar@gmail.com>]
license = "MIT/Apache-2.0"
build = "build.rs"

[features]
default = ["with_syntex"]
nightly = ["serde_macros"]
with_syntex = ["serde_codegen"]

[build-dependencies]
syntex = { version = "*", optional = true }
serde_codegen = { version = "*", optional = true }

[dependencies]
serde = "*"
serde_macros = { version = "*", optional = true }
```

Then the `build.rs` is changed to optionally expand the macros in our template:

```rust
#[cfg(feature = "with_syntex")]
mod inner {
    extern crate syntex;
    extern crate serde_codegen;

    use std::env;
    use std::path::Path;

    pub fn main() {
        let out_dir = env::var_os("OUT_DIR").unwrap();

        let src = Path::new("src/lib.rs.in");
        let dst = Path::new(&out_dir).join("lib.rs");

        let mut registry = syntex::Registry::new();

        serde_codegen::register(&mut registry);
        registry.expand("", &src, &dst).unwrap();
    }
}

#[cfg(not(feature = "with_syntex"))]
mod inner {
    pub fn main() {}
}

pub fn main() {
    inner::main()
}
```

Finally, `src/lib.rs` is updated to:

```rust
#[cfg_attr(feature = "nightly", feature(plugin))]
#[cfg_attr(feature = "nightly", plugin(serde_macros))]

extern crate serde;

#[cfg(feature = "nightly")]
include!("lib.rs.in");

#[cfg(feature = "with_syntex")]
include!(concat!(env!("OUT_DIR"), "/lib.rs"));
```

Then most development can happen with using the Nightly Rust and
`cargo build --no-default-features --features nightly` for better error
messages, but downstream consumers can use Stable Rust without worry.

Downside 2: Macros in Macros
----------------------------

Syntex can only expand macros inside macros it knows about, and it doesn't know
about the builtin macros. This is because a lot of the stable macros are using
unstable features under the covers. So unfortunately if you're using a library
like the quasiquoting library [quasi](https://github.com/erickt/rust-quasi),
you cannot write:

```rust
let exprs = vec![quote_expr!(cx, 1 + 2)];
```

Instead you have to pull out the syntex macros into a separate variable:

```rust
let expr = quote_expr!(cx, 1 + 1);
let exprs = vec![expr];
```

Downside 3: Compile Times
-------------------------

Syntex can take a while to compile. It may be possible to optimize this, but
that may be difficult while keeping compatibility with `libsyntax`.

---

That's `v0.4.0`. I hope you enjoy it! Please let me know if you run into any
[problems](https://github.com/serde-rs/serde/issues).

Release Notes
-------------

Here are other things that came with this version:

* Added field annotation to enable renaming fields for different backends
  [#69](https://github.com/serde-rs/serde/pull/69). For example:

```rust
struct Point {
  #[serde(rename="X")]
  x: u32,

  #[serde(rename(json="the-x", xml="X")]
  y: u32,
}
```

* Faster JSON string parsing [#71](https://github.com/serde-rs/serde/pull/71).
* Add a `LineColIterator` that tracks line and column information for
	deserializers [#58](https://github.com/serde-rs/serde/pull/58).
* Improved bytestring support [#72](https://github.com/serde-rs/serde/pull/72)
* Changed `de::PrimitiveVisitor` to also depend on `FromStr`
  [#70](https://github.com/serde-rs/serde/pull/70)
* Added impls for fixed sized arrays with 1 to 32 elements
  [#74](https://github.com/serde-rs/serde/pull/74)
* Added `json::Value::lookup`, that allows values to be extracted with
  `value.lookup("foo.bar.baz")` [#76](https://github.com/serde-rs/serde/pull/76)

Bug Fixes:

* Make sure that -0.0 gets serialized as "-0.0"
  [f0c87fb](https://github.com/serde-rs/serde/commit/f0c87fb).
* Missing field errors displayed original field name instead of renamed
  [#64](https://github.com/serde-rs/serde/pull/64).
* Fixed handling json integer overflow

A special thanks to everyone that helped with this release:

* Alex Crichton
* Andrew Poelstra
* Corey Farwell
* Hugo Duncan
* Kang Seonghoon
* Mikhail Borisov
* Oliver Schneider
* Sebastian Thiel
* Steven Fackler
* Thomas Bahn
* derhaskell
