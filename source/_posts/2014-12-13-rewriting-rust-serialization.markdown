---
layout: post
title: "Rewriting Rust Serialization, Part 3: Serde"
date: 2014-12-13 13:42:18 -0800
comments: true
categories: [rust, serialization]
---

There's been a long digression over the past month ([possible kernel bugs](),
[benchmarking Writers](), and [don't believe in magic, folks]()), but I'm back
into serialization. Woo! Here's
[part 1](http://erickt.github.io/blog/2014/10/28/serialization/) and
[part 2](http://erickt.github.io/blog/2014/11/03/performance/)), Rust's
[part 2.1](http://erickt.github.io/blog/2014/11/03/performance/)), Rust's
[part 2.2](http://erickt.github.io/blog/2014/11/03/performance/)) if you need
to catch up.

So `libserialize` has some pretty serious downsides. It's slow, it's got this
weird recursive closure thing going on, and it can't even represent enum types
like a `serialize::json::Json`. We need a new solution, and while I was at it,
we ended up with two: [serde]() and [serde2](). Both are different approaches
to trying to address these problems. The biggest one being the type
representation problem.

## Serde Version 1

### Deserialization

I want to start with deserialization first, as that's really the interesting
bit. To repeat myself a little bit from
[part 1](https://erickt.github.io/blog/2014/10/28/serialization/),
here is a generic json `Value` enum:

```rust
pub enum Value {
    I64(i64),
    U64(u64),
    F64(f64),
    String(String),
    Boolean(bool),
    Array(Vec<Value>),
    Object(TreeMap<String, Value>),
    Null,
}
```

To deserialize a string like `[1, true]` into
`Array(vec![I64(1), Boolean(true)])`, we need to peek at one character ahead
(ignoring whitespace) in order to discover what is the type of the next value.
We then can use that knowledge to pick the right variant, and parse the next
value correctly. While I haven't formally studied this stuff, I believe this
can be more formally stated as `Value` requires at least a LL(1) grammar,
but since `libserialize` supports no lookahead, so at most it can handle LL(0)
grammars.

Since I was thinking of this problem in terms of grammars, I wanted to take a
page out of their book and implement generic deserialization in this style.
`serde::de::Deserializer`s are then an `Iterator<serde::de::Token>` lexer that
produces a token stream, and `serde::de::Deserialize`s are a parser that
consumes this stream to produce a value. Here's `serde::de::Token`, which can
represent nearly all the rust types:

```rust
pub enum Token {
    Null,
    Bool(bool),
    Int(int),
    I8(i8),
    I16(i16),
    I32(i32),
    I64(i64),
    Uint(uint),
    U8(u8),
    U16(u16),
    U32(u32),
    U64(u64),
    F32(f32),
    F64(f64),
    Char(char),
    Str(&'static str),
    String(String),

    Option(bool),     // true if the option has a value

    TupleStart(uint), // estimate of the number of values

    StructStart(
        &'static str, // the struct name
        uint,         // estimate of the number of (string, value) pairs
    ),

    EnumStart(
        &'static str, // the enum name
        &'static str, // the variant name
        uint          // estimate of the number of values
    ),

    SeqStart(uint), // number of values

    MapStart(uint), // number of (value, value) pairs

    End,
}
```

The `serde::de::Deserialize` stream must generate tokens that follow this
grammar:

```antlr
value ::= Null
        | Bool
        | Int
        | ...
        | option
        | tuple
        | struct
        | enum
        | sequence
        | map
        ;

option ::= Option value
         | Option
         ;

tuple := TupleStart value* End;

struct := StructStart (Str value)* End;

enum := EnumStart value* End;

sequence := SeqStart value* End;

map := MapStart (value value)* End;
```

For performance reasons, there is no separator in the compound grammar.

Finishing up this section are the actual traits, `Deserialize` and `Deserializer`:

```rust
pub trait Deserialize<D: Deserializer<E>, E> {
    fn deserialize(d: &mut D) -> Result<Self, E> {
        let token = try!(d.expect_token());
        Deserialize::deserialize_token(d, token)
    }

    fn deserialize_token(d: &mut D, token: Token) -> Result<Self, E>;
}

pub trait Deserializer<E>: Iterator<Result<Token, E>> {
    /// Called when a `Deserialize` expected more tokens, but the
    /// `Deserializer` was empty.
    fn end_of_stream_error(&mut self) -> E;

    /// Called when a `Deserializer` was unable to properly parse the stream.
    fn syntax_error(&mut self, token: Token, expected: &'static [TokenKind]) -> E;

    /// Called when a named structure or enum got a name that it didn't expect.
    fn unexpected_name_error(&mut self, token: Token) -> E;

    /// Called when a value was unable to be coerced into another value.
    fn conversion_error(&mut self, token: Token) -> E;

    /// Called when a `Deserialize` structure did not deserialize a field
    /// named `field`.
    fn missing_field<
        T: Deserialize<Self, E>
    >(&mut self, field: &'static str) -> Result<T, E>;

    /// Called when a `Deserialize` has decided to not consume this token.
    fn ignore_field(&mut self, _token: Token) -> Result<(), E> {
        let _: IgnoreTokens = try!(Deserialize::deserialize(self));
        Ok(())
    }

    #[inline]
    fn expect_token(&mut self) -> Result<Token, E> {
        match self.next() {
            Some(Ok(token)) => Ok(token),
            Some(Err(err)) => Err(err),
            None => Err(self.end_of_stream_error()),
        }
    }

    ...
}
```

The `Deserialize` trait is kept pretty slim, and is how lookahead is
implemented. `Deserializer` is an enhanced `Iterator<Result<Token, E>>`, with
many helpful default methods. Here are them in action. First we'll start with
what's probably the simplest `Deserializer`, which just wraps a prebuilt
`Vec<Token>`:


```rust
enum Error {
    EndOfStream,
    SyntaxError(Vec<TokenKind>),
    UnexpectedName,
    ConversionError,
    MissingField(&'static str),
}

struct TokenDeserializer<Iter> {
    tokens: Iter,
}

impl<Iter: Iterator<Token>> TokenDeserializer<Iter> {
    fn new(tokens: Iter) -> TokenDeserializer<Iter> {
        TokenDeserializer {
            tokens: tokens,
        }
    }
}

impl<Iter: Iterator<Token>> Iterator<Result<Token, Error>> for TokenDeserializer<Iter> {
    fn next(&mut self) -> option::Option<Result<Token, Error>> {
        match self.tokens.next() {
            None => None,
            Some(token) => Some(Ok(token)),
        }
    }
}

impl<Iter: Iterator<Token>> Deserializer<Error> for TokenDeserializer<Iter> {
    fn end_of_stream_error(&mut self) -> Error {
        Error::EndOfStream
    }

    fn syntax_error(&mut self, _token: Token, expected: &[TokenKind]) -> Error {
        Error::SyntaxError(expected.to_vec())
    }

    fn unexpected_name_error(&mut self, _token: Token) -> Error {
        Error::UnexpectedName
    }

    fn conversion_error(&mut self, _token: Token) -> Error {
        Error::ConversionError
    }

    #[inline]
    fn missing_field<
        T: Deserialize<TokenDeserializer<Iter>, Error>
    >(&mut self, field: &'static str) -> Result<T, Error> {
        Err(Error::MissingField(field))
    }
}
```

Overall it should be pretty straight forward. As usual, error handling makes
things a bit noisier, but hopefully it's not too onerous. Next is a
`Deserialize` for `bool`:

```rust
impl<D: Deserializer<E>, E> Deserialize<D, E> for bool {
    #[inline]
    fn deserialize_token(d: &mut D, token: Token) -> Result<bool, E> {
        d.expect_bool(token)
    }
}

pub trait Deserializer<E>: Iterator<Result<Token, E>> {
    ...

    #[inline]
    fn expect_bool(&mut self, token: Token) -> Result<bool, E> {
        match token {
            Token::Bool(value) => Ok(value),
            token => {
                static EXPECTED_TOKENS: &'static [TokenKind] = &[
                    TokenKind::BoolKind,
                ];
                Err(self.syntax_error(token, EXPECTED_TOKENS))
            }
        }
    }


    ...
}
```

Simple! Sequences are a bit more tricky. Here's `Deserialize` a `Vec<T>`. We
use a helper adaptor `SeqDeserializer` to deserialize from all types that
implement `FromIterator`:

```rust
impl<
    D: Deserializer<E>,
    E,
    T: Deserialize<D ,E>
> Deserialize<D, E> for Vec<T> {
    #[inline]
    fn deserialize_token(d: &mut D, token: Token) -> Result<Vec<T>, E> {
        d.expect_seq(token)
    }
}

pub trait Deserializer<E>: Iterator<Result<Token, E>> {
    ...

    #[inline]
    fn expect_seq<
        T: Deserialize<Self, E>,
        C: FromIterator<T>
    >(&mut self, token: Token) -> Result<C, E> {
        let len = try!(self.expect_seq_start(token));
        let mut err = None;

        let collection: C = {
            let d = SeqDeserializer {
                d: self,
                len: len,
                err: &mut err,
            };

            d.collect()
        };

        match err {
            Some(err) => Err(err),
            None => Ok(collection),
        }
    }

    ...
}

struct SeqDeserializer<'a, D: 'a, E: 'a> {
    d: &'a mut D,
    len: uint,
    err: &'a mut Option<E>,
}

impl<
    'a,
    D: Deserializer<E>,
    E,
    T: Deserialize<D, E>
> Iterator<T> for SeqDeserializer<'a, D, E> {
    #[inline]
    fn next(&mut self) -> option::Option<T> {
        match self.d.expect_seq_elt_or_end() {
            Ok(next) => {
                self.len -= 1;
                next
            }
            Err(err) => {
                *self.err = Some(err);
                None
            }
        }
    }

    #[inline]
    fn size_hint(&self) -> (uint, option::Option<uint>) {
        (self.len, Some(self.len))
    }
}
```

Last is a struct deserializer. This relies on a simple state machine in order
to deserialize from out of order maps:

```rust
struct Inner {
    a: (),
    b: uint,
    c: TreeMap<String, Option<char>>,
}

impl<
    D: Deserializer<E>,
    E
> Deserialize<D, E> for Inner {
    #[inline]
    fn deserialize_token(d: &mut D, token: Token) -> Result<Inner, E> {
        try!(d.expect_struct_start(token, "Inner"));

        let mut a = None;
        let mut b = None;
        let mut c = None;

        static FIELDS: &'static [&'static str] = &["a", "b", "c"];

        loop {
            let idx = match try!(d.expect_struct_field_or_end(FIELDS)) {
                Some(idx) => idx,
                None => { break; }
            };

            match idx {
                Some(0) => { a = Some(try!(d.expect_struct_value())); }
                Some(1) => { b = Some(try!(d.expect_struct_value())); }
                Some(2) => { c = Some(try!(d.expect_struct_value())); }
                Some(_) => unreachable!(),
                None => { let _: IgnoreTokens = try!(Deserialize::deserialize(d)); }
            }
        }

        Ok(Inner { a: a.unwrap(), b: b.unwrap(), c: c.unwrap() })
    }
}
```

It's more complicated than `libserialize`'s struct parsing, but it performs
much better because it can handle out of order maps without buffering tokens.

### Serialization

Serialization's story is a much simpler one. Conceptually
`serde::ser::Serializer`/`serde::ser::Serialize` are inspired by the
deserialization story, but we don't need the tagged tokens because we already
know the types. Here are the traits:

```rust
pub trait Serialize<S: Serializer<E>, E> {
    fn serialize(&self, s: &mut S) -> Result<(), E>;
}

pub trait Serializer<E> {
    fn serialize_null(&mut self) -> Result<(), E>;

    fn serialize_bool(&mut self, v: bool) -> Result<(), E>;

    #[inline]
    fn serialize_int(&mut self, v: int) -> Result<(), E> {
        self.serialize_i64(v as i64)
    }

    #[inline]
    fn serialize_i8(&mut self, v: i8) -> Result<(), E> {
        self.serialize_i64(v as i64)
    }

    #[inline]
    fn serialize_i16(&mut self, v: i16) -> Result<(), E> {
        self.serialize_i64(v as i64)
    }

    #[inline]
    fn serialize_i32(&mut self, v: i32) -> Result<(), E> {
        self.serialize_i64(v as i64)
    }

    #[inline]
    fn serialize_i64(&mut self, v: i64) -> Result<(), E>;

    #[inline]
    fn serialize_uint(&mut self, v: uint) -> Result<(), E> {
        self.serialize_u64(v as u64)
    }

    #[inline]
    fn serialize_u8(&mut self, v: u8) -> Result<(), E> {
        self.serialize_u64(v as u64)
    }

    #[inline]
    fn serialize_u16(&mut self, v: u16) -> Result<(), E> {
        self.serialize_u64(v as u64)
    }

    #[inline]
    fn serialize_u32(&mut self, v: u32) -> Result<(), E> {
        self.serialize_u64(v as u64)
    }

    #[inline]
    fn serialize_u64(&mut self, v: u64) -> Result<(), E>;

    #[inline]
    fn serialize_f32(&mut self, v: f32) -> Result<(), E> {
        self.serialize_f64(v as f64)
    }

    fn serialize_f64(&mut self, v: f64) -> Result<(), E>;

    fn serialize_char(&mut self, v: char) -> Result<(), E>;

    fn serialize_str(&mut self, v: &str) -> Result<(), E>;

    fn serialize_tuple_start(&mut self, len: uint) -> Result<(), E>;
    fn serialize_tuple_elt<
        T: Serialize<Self, E>
    >(&mut self, v: &T) -> Result<(), E>;
    fn serialize_tuple_end(&mut self) -> Result<(), E>;

    fn serialize_struct_start(&mut self, name: &str, len: uint) -> Result<(), E>;
    fn serialize_struct_elt<
        T: Serialize<Self, E>
    >(&mut self, name: &str, v: &T) -> Result<(), E>;
    fn serialize_struct_end(&mut self) -> Result<(), E>;

    fn serialize_enum_start(&mut self, name: &str, variant: &str, len: uint) -> Result<(), E>;
    fn serialize_enum_elt<
        T: Serialize<Self, E>
    >(&mut self, v: &T) -> Result<(), E>;
    fn serialize_enum_end(&mut self) -> Result<(), E>;

    fn serialize_option<
        T: Serialize<Self, E>
    >(&mut self, v: &Option<T>) -> Result<(), E>;

    fn serialize_seq<
        T: Serialize<Self, E>,
        Iter: Iterator<T>
    >(&mut self, iter: Iter) -> Result<(), E>;

    fn serialize_map<
        K: Serialize<Self, E>,
        V: Serialize<Self, E>,
        Iter: Iterator<(K, V)>
    >(&mut self, iter: Iter) -> Result<(), E>;
}
```

There are many default methods, so only a handful of implementations need to be
specified. Now lets look at how they are used. Here's a simple
`AssertSerializer` that I use in my test suite to make sure I'm serializing
properly:

```rust
struct AssertSerializer<Iter> {
    iter: Iter,
}

impl<'a, Iter: Iterator<Token<'a>>> AssertSerializer<Iter> {
    fn new(iter: Iter) -> AssertSerializer<Iter> {
        AssertSerializer {
            iter: iter,
        }
    }

    fn serialize<'b>(&mut self, token: Token<'b>) -> Result<(), Error> {
        let t = match self.iter.next() {
            Some(t) => t,
            None => { panic!(); }
        };

        assert_eq!(t, token);

        Ok(())
    }
}

impl<'a, Iter: Iterator<Token<'a>>> Serializer<Error> for AssertSerializer<Iter> {
    fn serialize_null(&mut self) -> Result<(), Error> {
        self.serialize(Token::Null)
    }
    fn serialize_bool(&mut self, v: bool) -> Result<(), Error> {
        self.serialize(Token::Bool(v))
    }
    fn serialize_int(&mut self, v: int) -> Result<(), Error> {
        self.serialize(Token::Int(v))
    }
    ...
}
```

Implementing `Serialize` for values follows the same pattern. Here's `bool`:

```
impl<S: Serializer<E>, E> Serialize<S, E> for bool {
    #[inline]
    fn serialize(&self, s: &mut S) -> Result<(), E> {
        s.serialize_bool(*self)
    }
}
```

`Vec<T>`:

```rust
impl<
    S: Serializer<E>,
    E,
    T: Serialize<S, E>
> Serialize<S, E> for Vec<T> {
    #[inline]
    fn serialize(&self, s: &mut S) -> Result<(), E> {
        s.serialize_seq(self.iter())
    }
}

pub trait Serializer<E> {
    ...

    fn serialize_seq<
        T: Serialize<AssertSerializer<Iter>, Error>,
        SeqIter: Iterator<T>
    >(&mut self, mut iter: SeqIter) -> Result<(), Error> {
        let (len, _) = iter.size_hint();
        try!(self.serialize(Token::SeqStart(len)));
        for elt in iter {
            try!(elt.serialize(self));
        }
        self.serialize(Token::SeqEnd)
    }

    ...
}
```

And structs:

```rust
struct Inner {
    a: (),
    b: uint,
    c: TreeMap<String, Option<char>>,
}

impl<
  S: Serializer<E>,
  E
> Serialize<S, E> for Inner {
    fn serialize(&self, s: &mut S) -> Result<(), E> {
        try!(s.serialize_struct_start("Inner", 2u));
        try!(s.serialize_struct_elt("a", &self.a));
        try!(s.serialize_struct_elt("b", &self.b));
        try!(s.serialize_struct_elt("c", &self.c));
        s.serialize_struct_end()
    }
}

```

Much simpler than deserialization.

## Performance















--

| language | library           | serialization (MB/s) | deserialization (MB/s) |
| -------- | ----------------- | -------------------- | ---------------------- |
| Rust     | serde::json       | 348                  | 69                     |





serialization:

| language | library         | speed        |
| -------- | --------------- | ------------ |
| C++      | rapidjson       | 243 MB/s     |
| Go       | encoding/json   | 58.14 MB/s   |
| Go       | ffjson          |              |
| Go       | goprotobuf      | 136.50 MB/s  |
| Go       | gogoprotobuf    | 464.36 MB/s  |
| Go       | go-capnproto    | 2919.85 MB/s |
| Rust     | serialize::json | 83 MB/s      |

deserialization:

| language | library                  | speed        |
| -------- | ------------------------ | ------------ |
| C++      | rapidjson                | 243 MB/s     |
| Go       | encoding/json            | 58.14 MB/s   |
| Go       | ffjson                   |              |
| Go       | goprotobuf               | 136.50 MB/s  |
| Go       | gogoprotobuf             | 464.36 MB/s  |
| Go       | go-capnproto             | 543.28 MB/s  |
| Go       | go-capnproto (zero-copy) | 1328.47 MB/s |
| Rust     | serialize::json          | 20 MB/s      |


 to both to Rust and C++'s


I've ported over Cloudflare's Go language
[Goser](https://github.com/cloudflare/goser) to both to Rust and C++'s
[rapidjson](https://github.com/miloyip/rapidjson)

 benchmark to see how we
performed against other

 on a next generation generic serialization library
[serde](https://github.com/erickt/rust-serde). It tries to clean up some of the
ugliness and performance issues in Rust's current


It' actually contains two approaches that I've been experimenting with.



, which is my rewrite of the rust
serialization framework. In a benchmark I ported from [Cloudflare's
Goser](https://github.com/cloudflare/goser) benchmarks,

The contenders:

|language   |library          |speed                                                                            |
|-----------|-----------------|--------------                                                                   |
|C++        |JSON             |[rapidjson](https://github.com/miloyip/rapidjson)                                |
|Go         |JSON             |[encoding/json](http://golang.org/pkg/encoding/json)                             |
|Go         |JSON             |[ffjson](https://github.com/pquerna/json)                                        |
|Go         |Protocol Buffers |[goprotobuf](http://code.google.com/p/goprotobuf/)                               |
|Go         |Protocol Buffers |[gogoprotobuf](http://code.google.com/p/goprotobuf/)                             |
|Go         |Cap'n Proto      |[go-capnproto](http://code.google.com/p/gogoprotobuf/)                           |
|Rust       |JSON             |[serialize::json](http://doc.rust-lang.org/serialize/json/)                      |
|Rust       |JSON             |[serde::json](https://github.com/erickt/rust-serde/tree/master/src/json/)        |
|Rust       |JSON             |[serde2::json](https://github.com/erickt/rust-serde/tree/master/serde2/src/json/)|

  Anyway, my main project this
past year has been

serialization:

| language    | library           | speed          |
| ----------- | ----------------- | -------------- |
| C++         | rapidjson         | 243 MB/s       |
| Go          | encoding/json     | 58.14 MB/s     |
| Go          | ffjson            |                |
| Go          | goprotobuf        | 136.50 MB/s    |
| Go          | gogoprotobuf      | 464.36 MB/s    |
| Go          | go-capnproto      | 2919.85 MB/s   |
| Rust        | serialize::json   | 83 MB/s        |
| Rust        | serde::json       | 253 MB/s       |
| Rust        | serde2::json      | 215 MB/s       |
| foo         | bar               | baz            |

deserialization:

| language  | library                  | speed        |
|-----------|--------------------------|--------------|
| C++       | rapidjson                | 243 MB/s     |
| Go        | encoding/json            | 58.14 MB/s   |
| Go        | ffjson                   |              |
| Go        | goprotobuf               | 136.50 MB/s  |
| Go        | gogoprotobuf             | 464.36 MB/s  |
| Go        | go-capnproto             | 543.28 MB/s  |
| Go        | go-capnproto (zero-copy) | 1328.47 MB/s |
| Rust      | serialize::json          | 20 MB/s      |
| Rust      | serde::json              | 47 MB/s      |
| Rust      | serde2::json             | -            |
