---
layout: post
title: "Rewriting Rust Serialization, Part 3: Serde"
date: 2014-12-09 07:42:18 -0800
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
like a `json::Json`. We need a new solution, and while I was at it, we ended up
with two: [serde]() and [serde2](). Both are different approaches to trying to
address these problems. The biggest one being the type representation problem.

## serde version 1

To possibly repeat myself a little bit from
[part 1](https://erickt.github.io/blog/2014/10/28/serialization/), here's the
`json::Json` type:

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

To deserialize a string like `[1, true]` into
`List(vec![I64(1), Boolean(true)])`, we need to peek at one character ahead
(ignoring whitespace) in order to pick which type we should be deserializing
into. While I haven't formally studied this stuff, I believe this can be more
formally stated as `json::Json` requires at least a LL(1) grammar, but since
`libserialize` supports no lookahead, so at most it can handle LL(0) grammars.

Since I was thinking of this problem in terms of parsers, I took a page out of
their book and wrote [serde]() to produce a tagged token stream of these
values:

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
    String(string::String),

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

The primitives are pretty self explanatory, the main emphasis is on compound
types. I wanted to allow consumers of the token stream to not have to batch up
compound values into something like `Seq(Vec<Box<Token>>)`, so I let them use a
protocol where a compound type has a start, a series of values, and an end. You
can see that expressed with this grammar:

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

serde deserializers then are literal `Iterator`s over a stream of these
tokens. Here is the simplest Deserializer, one that just produces a
preallocated stream of tokens:

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

Overall, pretty simple, only the `Deserializer` error handling needs to be
implemented (I'm considering even moving those methods into an
`serde::de::Error` trait). There are a number of default methods on
`Deserializer` that I'll get to in a moment.

The other side of the handshake is the `Deserialize` trait, which is where you
can see our lookahead support:

```rust
pub trait Deserialize<D: Deserializer<E>, E> {
    fn deserialize(d: &mut D) -> Result<Self, E> {
        let token = try!(d.expect_token());
        Deserialize::deserialize_token(d, token)
    }

    fn deserialize_token(d: &mut D, token: Token) -> Result<Self, E>;
}
```

We peel off one token from the stream using the and pass it to the virtual
`deserialize_token` to actually extract the value.

Now it's a bit of a pain to have to manually parse these tokens, so
`Deserializer` comes with a number of default helper methods. We saw
`expect_token()` previously. Well that just provides error handling if we hit the end of the stream:

```rust
fn expect_token(&mut self) -> Result<Token, E> {
		match self.next() {
				Some(Ok(token)) => Ok(token),
				Some(Err(err)) => Err(err),
				None => Err(self.end_of_stream_error()),
		}
}
```

Primitive types get the same treatment. Here's how to parse `()`:

```rust
impl<
		D: Deserializer<E>,
		E
> Deserialize<D, E> for () {
		fn deserialize_token(d: &mut D, token: Token) -> Result<(), E> {
				d.expect_null(token)
		}
}
```

`Deserializer::expect_null()` then handles parsing from a raw `Null` token,
zero-length tuples or sequences:

```rust
fn expect_null(&mut self, token: Token) -> Result<(), E> {
		match token {
				Token::Null => Ok(()),
				Token::TupleStart(_) | Token::SeqStart(_) => {
						match try!(self.expect_token()) {
								Token::End => Ok(()),
								token => {
										static EXPECTED_TOKENS: &'static [TokenKind] = &[
												TokenKind::EndKind,
										];
										Err(self.syntax_error(token, EXPECTED_TOKENS))
								}
						}
				}
				token => {
						static EXPECTED_TOKENS: &'static [TokenKind] = &[
								TokenKind::NullKind,
								TokenKind::TupleStartKind,
								TokenKind::SeqStartKind,
						];
						Err(self.syntax_error(token, EXPECTED_TOKENS))
				}
		}
}
```

Structs and variants are where things get a bit complicated. Back in
`libserialize`, we had no support for deserializing from out-of-order map (like
a JSON object) into a struct. What you would have to do is first load all the
values from the map into a generic structure like `serialize::json::Json`, then
look up each field from the map and deserialize from that buffered value. You
can imagine this can be quite slow. So in `serde`, 

 stream. S

buffer
all the values in a map into a `HashMap`, then pop off the values out of the
map 

 Instead, it's possible to make a simple
state machine to 











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
