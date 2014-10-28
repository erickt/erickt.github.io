---
layout: post
title: "Rewriting Rust Serialization, Part 1"
date: 2014-10-28 08:52:18 -0700
comments: true
categories: [rust, serialization]
---

Hello everybody! It's been, what, *two* years since I last blogged? Not my best
performance, I'm sorry to say. So for all of my 3 pageviews that are probably
bots, I appologize for such a long delay on updating my blog. I got to say I've
been pretty inspired by the great [Julia Evans](http://jvns.ca/) (who I hope we
can someday get back to working on rust stuff). She's an epic blogger, and I
hope I can get somewhere near that speed.

Anyway, on to the post. My main on-again-off-again project this past year has
been working Rust's generic [serialize](http://doc.rust-lang.org/serialize/)
library. If you haven't played with it yet, it's really nifty. It's a generic
framework that allows a generic `Encoder` serialize a generic `Encodable`, and
the inverse with `Decoder` and `Decodable`. This allows you to write just one
`Encodable` impl that can transparently work with our
[json](http://doc.rust-lang.org/serialize/) library,
[msgpack](https://github.com/mneumann/rust-msgpack),
[toml](https://github.com/alexcrichton/toml-rs), and etc. It's simple to use
too in most cases as you can use `#[deriving(Encodable, Decodable)]` to
automatically create a implementation for your type. Here's an example:

```rust
extern crate serialize;

use serialize::json;

#[deriving(Encodable, Decodable, Show)]
struct Employee {
    name: String,
}

#[deriving(Encodable, Decodable, Show)]
struct Company {
    employees: Vec<Employee>,
}

fn main() {
    let company = Company {
        employees: vec![
            Employee { name: "Dan".to_string() },
            Employee { name: "Erin".to_string() },
            Employee { name: "Jeff".to_string() },
            Employee { name: "Spencer".to_string() },
        ],
    };

    let s = json::encode(&company);
    let company: Company = json::decode(s.as_slice()).unwrap();
}
```

There are some downsides to serialize though. Manually implementing can be a
bit of a pain. Here's the example from before:

```rust
impl<S: Encoder<E>, E> Encodable<S, E> for Employee {
    fn encode(&self, s: &mut S) -> Result<(), E> {
        match *self {
            Employee { name: ref name } => {
                s.emit_struct("Employee", 1u, |s| {
                    s.emit_struct_field("name", 0u, |s| name.encode(s))
                })
            }
        }
    }
}

impl<D: Decoder<E>, E> Decodable<D, E> for Employee {
    fn decode(d: &mut D) -> Result<Employee, E> {
        d.read_struct("Employee", 1u, |d| {
            Ok(Employee {
                name: {
                    try!(d.read_struct_field("name", 0u, |d| {
                        Decodable::decode(d)
                    }))
                }
            })
        })
    }
}
```

As you can see, parsing compound structures requires these recursive closure
calls in order to perform the handshake between the `Encoder` and the
`Encodable`. A couple people have run into bugs in the past where they didn't
implement this pattern, which results in some confusing bugs. Furthermore, LLVM
isn't great at inlining these recursive calls, so `serialize` impls tend to not
perform well.

That's not the worst of it though. The real problem is that there are types
that can implement `Encodable`, there's no way to write a `Decodable`
implementation. They're pretty common too. For example, the
`serialize::json::Json` type:

```rust
pub enum Json {
    I64(i64),
    U64(u64),
    F64(f64),
    String(string::String),
    Boolean(bool),
    List(JsonList),
    Object(JsonObject),
    Null,
}

pub type JsonList = Vec<Json>;
pub type JsonObject = TreeMap<string::String, Json>;
```

The `Json` value can represent any value that's in a JSON string. Implied in
this is the notion that the `Decodable` has to look ahead to see what the next
value is so it can decide which `Json` variant to construct. Unfortunately our
current `Decoder` infrastructure doesn't support lookahead. The way the
`Decoder`/`Decodable` handshake works is essentially:

 * `Decodable` asks for a struct named `"Employee"`.
 * `Decodable` asks for a field named `"name"`.
 * `Decodable` asks for a value of type `String`.
 * `Decodable` asks for a field named `"age"`.
 * `Decodable` asks for a value of type `uint`.
 * ...

Any deviation from this pattern results in an error. There isn't a way for the
`Decodable` to ask what is the type of the next value, so this is why we
serialize generic enums by explicitly tagging the variant, as in:

```
extern crate serialize;

use serialize::json;

#[deriving(Encodable, Decodable, Show)]
enum Animal {
    Dog(uint),
    Frog(String, uint),
}

fn main() {
    let animal = Frog("Frank".to_string(), 349);

    let s = json::encode(&animal);

    println!("{}", s);
    // prints {"variant":"Frog","fields":["Frank",349]}
}
```

That's probably good enough for now. In my next post I'll go into in my
approach to fix this in [serde](https://github.com/erickt/rust-serde).
