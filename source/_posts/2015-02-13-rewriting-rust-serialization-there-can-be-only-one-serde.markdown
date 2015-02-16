---
layout: post
title: "Rewriting Rust Serialization: Part 4: Serde2 is ready!"
date: 2015-02-16 07:32:54 -0800
comments: true
categories: [rust, serialization, serde]
---

It's been a while, hasn't it? Here's
[part 1](http://erickt.github.io/blog/2014/10/28/serialization/),
[part 2](http://erickt.github.io/blog/2014/11/03/performance/),
[part 2.1](http://erickt.github.io/blog/2014/11/03/performance/),
[part 2.2](http://erickt.github.io/blog/2014/11/03/performance/),
[part 3](http://erickt.github.io/blog/2014/12/13/rewriting-rust-serialization/), and
[part 3.1](http://erickt.github.io/blog/2014/12/13/performance-digression/)
if you want to catch up.

## Serde Version 2

Well it's a long time coming, but serde2 is finally in a mostly usable
position! If you recall from
[part 3](http://erickt.github.io/blog/2014/12/13/rewriting-rust-serialization/),
one of the problems with serde1 is that we're paying a lot for tagging our
types, and it's really hurting us on the deserialization side of things. So
there's one other pattern that we can use that allows for lookahead that
doesn't need tags: visitors. A year or so ago I rewrote our generic hashing
framework to use the visitor pattern to great success. `serde2` came out of
experiments to see if I could do the same thing here. It turned out that it was
a really elegant approach.

### Serialize

It all starts with a type that we want to serialize:

```rust
pub trait Serialize {
    fn visit<
        V: Visitor,
    >(&self, visitor: &mut V) -> Result<V::Value, V::Error>;
}
```

(Aside: while I'd rather use `where` here for this type parameter, that would
force me to write `<V as Visitor>::Value>` due to
[#20300](https://github.com/rust-lang/rust/issues/20300)).

This `Visitor` trait then looks like:

```rust
pub trait Visitor {
    type Value;
    type Error;

    fn visit_unit(&mut self) -> Result<Self::Value, Self::Error>;

    #[inline]
    fn visit_named_unit(&mut self, _name: &str) -> Result<Self::Value, Self::Error> {
        self.visit_unit()
    }


    fn visit_bool(&mut self, v: bool) -> Result<Self::Value, Self::Error>;

    ...
}
```

So the implementation for a `bool` then looks like:

```rust
impl Serialize for bool {
    #[inline]
    fn visit<
        V: Visitor,
    >(&self, visitor: &mut V) -> Result<V::Value, V::Error> {
        visitor.visit_bool(*self)
    }
}
```

Things get more interesting when we get to compound structures like a sequence.
Here's `Visitor` again. It needs to both be able to visit the overall structure
as well as each item:

```rust
    ...

    fn visit_seq<V>(&mut self, visitor: V) -> Result<Self::Value, Self::Error>
        where V: SeqVisitor;

    fn visit_seq_elt<T>(&mut self,
                        first: bool,
                        value: T) -> Result<Self::Value, Self::Error>
        where T: Serialize;

    ...
}
```

We also have this `SeqVisitor` trait that the type to serialize provides. It
really just looks like an `Iterator`, but the type parameter has been moved to
the `visit` method so that it can return different types:

```rust
pub trait SeqVisitor {
    fn visit<
        V: Visitor,
    >(&mut self, visitor: &mut V) -> Result<Option<V::Value>, V::Error>;

    #[inline]
    fn size_hint(&self) -> (usize, Option<usize>) {
        (0, None)
    }
}
```

Finally, to implement this for a type like `&[T]` we create an
`Iterator`-to-`SeqVisitor` adaptor and pass it to the visitor, which then in
turn visits each item:

```rust
pub struct SeqIteratorVisitor<Iter> {
    iter: Iter,
    first: bool,
}

impl<T, Iter: Iterator<Item=T>> SeqIteratorVisitor<Iter> {
    #[inline]
    pub fn new(iter: Iter) -> SeqIteratorVisitor<Iter> {
        SeqIteratorVisitor {
            iter: iter,
            first: true,
        }
    }
}

impl<
    T: Serialize,
    Iter: Iterator<Item=T>,
> SeqVisitor for SeqIteratorVisitor<Iter> {
    #[inline]
    fn visit<
        V: Visitor,
    >(&mut self, visitor: &mut V) -> Result<Option<V::Value>, V::Error> {
        let first = self.first;
        self.first = false;

        match self.iter.next() {
            Some(value) => {
                let value = try!(visitor.visit_seq_elt(first, value));
                Ok(Some(value))
            }
            None => Ok(None),
        }
    }

    #[inline]
    fn size_hint(&self) -> (usize, Option<usize>) {
        self.iter.size_hint()
    }
}

impl<
    'a,
    T: Serialize,
> Serialize for &'a [T] {
    #[inline]
    fn visit<
        V: Visitor,
    >(&self, visitor: &mut V) -> Result<V::Value, V::Error> {
        visitor.visit_seq(SeqIteratorVisitor::new(self.iter()))
    }
}
```

`SeqIteratorVisitor` is publically exposed, so it should be easy to use it with
custom data structures. Maps follow the same pattern (and also expose
`MapIteratorVisitor`), but each item instead uses `visit_visit_map_elt(first,
key, value)`.  Tuples, struct tuples, and tuple enum variants are all really
just named sequences. Likewise, structs and struct enum variants are just named
maps.

Because struct implementations are so common, here's an example how to do it:

```rust
struct Point {
    x: i32,
    y: i32,
}

struct PointVisitor<'a> {
    state: u32,
    value: &'a Point,
}

impl<'a> MapVisitor for PointVisitor<'a> {
    fn visit<
        V: Visitor,
    >(&mut self, visitor: &mut V) -> Result<V::Value, V::Error> {
        match self.state {
            0 => {
                self.state += 1;
                Ok(Some(try!(visitor.visit_map_elt(true, "x", &self.x))))
            }
            1 => {
                self.state += 1;
                Ok(Some(try!(visitor.visit_map_elt(true, "y", &self.y))))
            }
            _ => Ok(None),
        }
    }
}

impl Serialize for Point {
    fn visit<
        V: Visitor,
    >(&self, visitor: &mut V) -> Result<V::Value, V::Error> {
        visit_named_map("Point", PointVisitor {
            state: 0,
            value: self,
        })
    }
}
```

Fortunately `serde2` also comes with a `#[derive_serialize]` macro so you don't
need to write this out by hand if you don't want to.

### Serializer

Now to actually build a serializer. We start with a trait:

```rust
pub trait Serializer {
    type Value;
    type Error;

    fn visit<T>(&mut self, value: &T) -> Result<Self::Value, Self::Error>
        where T: Serialize;
}
```

It's the responsibility of the serializer to create a visitor and then pass it
to the type. Oftentimes the serializer also implements `Visitor`, but it's not
required. Here's a snippet of the JSON serializer visitor:

```rust
struct Visitor<'a, W: 'a> {
    writer: &'a mut W,
}

impl<'a, W: Writer> Visitor for Visitor<'a, W> {
    type Value = ();
    type Error = io::Error;

    fn visit_unit(&mut self) -> IoResult<()> {
        self.writer.write_all(b"null")
    }

    #[inline]
    fn visit_bool(&mut self, value: bool) -> IoResult<()> {
        if value {
            self.writer.write_all(b"true")
        } else {
            self.writer.write_all(b"false")
        }
    }

    #[inline]
    fn visit_isize(&mut self, value: isize) -> IoResult<()> {
        write!(self.writer, "{}", value)
    }

    ...

    #[inline]
    fn visit_map<V>(&mut self, mut visitor: V) -> IoResult<()>
        where V: ser::MapVisitor,
    {
        try!(self.writer.write_all(b"{"));

        while let Some(()) = try!(visitor.visit(self)) { }

        self.writer.write_all(b"}")
    }

    #[inline]
    fn visit_map_elt<K, V>(&mut self, first: bool, key: K, value: V) -> IoResult<()>
        where K: ser::Serialize,
              V: ser::Serialize,
    {
        if !first {
            try!(self.writer.write_all(b","));
        }

        try!(key.visit(self));
        try!(self.writer.write_all(b":"));
        value.visit(self)
    }
}
```

Hopefully it is pretty straight forward.

## Deserialization

Now serialization is the easy part. Deserialization is where it always gets
more tricky. We follow a similar pattern as serialization. A deserializee
creates a visitor which accepts any type (most resulting in an error), and
passes it to a deserializer. This deserializer then extracts it's next value
from it's stream and passes it to the visitor, which then produces the actual
type.

It's achingly close to the same pattern between a serializer and a serializee,
but as hard as I tried, I couldn't unify the two. The error semantics are
different. In serialization, you want the serializer (which creates the
visitor) to define the error. In deserialization, you want the deserializer
which consumes the visitor to define the error.

Let's start first with `Error`. As opposed to serialization, when we're
deserializing we can error both in the `Deserializer` if there is a parse
error, or in the `Deserialize` if it's received an unexpected value. We do this
with an `Error` trait, which allows a deserializee to generically create the
few errors it needs:

```rust
pub trait Error {
    fn syntax_error() -> Self;

    fn end_of_stream_error() -> Self;

    fn missing_field_error(&'static str) -> Self;
}
```

Now the `Deserialize` trait, which looks similar to `Serialize`:

```rust
pub trait Deserialize {
    fn deserialize<
        D: Deserializer,
    >(deserializer: &mut D) -> Result<Self, D::Error>;
}
```

The `Visitor` also looks like the serialization `Visitor`, except for the
methods error by default.

```rust
pub trait Visitor {
    type Value;

    fn visit_bool<
        E: Error,
    >(&mut self, _v: bool) -> Result<Self::Value, E> {
        Err(Error::syntax_error())
    }

    fn visit_isize<
        E: Error,
    >(&mut self, v: isize) -> Result<Self::Value, E> {
        self.visit_i64(v as i64)
    }

    ...
}
```

Sequences and Maps are also a little different:

```rust
pub trait Visitor {
    ...

    fn visit_seq<
        V: SeqVisitor,
    >(&mut self, _visitor: V) -> Result<Self::Value, V::Error> {
        Err(Error::syntax_error())
    }

    fn visit_map<
        V: MapVisitor,
    >(&mut self, _visitor: V) -> Result<Self::Value, V::Error> {
        Err(Error::syntax_error())
    }

    ...
}

pub trait SeqVisitor {
    type Error: Error;

    fn visit<
        T: Deserialize,
    >(&mut self) -> Result<Option<T>, Self::Error>;

    fn end(&mut self) -> Result<(), Self::Error>;

    #[inline]
    fn size_hint(&self) -> (usize, Option<usize>) {
        (0, None)
    }
}

pub trait MapVisitor {
    type Error: Error;

    #[inline]
    fn visit<
        K: Deserialize,
        V: Deserialize,
    >(&mut self) -> Result<Option<(K, V)>, Self::Error> {
        match try!(self.visit_key()) {
            Some(key) => {
                let value = try!(self.visit_value());
                Ok(Some((key, value)))
            }
            None => Ok(None)
        }
    }

    fn visit_key<
        K: Deserialize,
    >(&mut self) -> Result<Option<K>, Self::Error>;

    fn visit_value<
        V: Deserialize,
    >(&mut self) -> Result<V, Self::Error>;

    fn end(&mut self) -> Result<(), Self::Error>;

    #[inline]
    fn size_hint(&self) -> (usize, Option<usize>) {
        (0, None)
    }
}
```

Here is an example struct deserializer. Structs are deserialized as a map, but
since maps are unordered, we need a simple state machine to extract the values.
In order to get the keys, we just create an enum for the fields, and a custom
deserializer to convert a string into a field without an allocation:

```rust
struct Point {
    x: i32,
    y: i32,
}

impl Deserialize for Point {
    fn deserialize<
        D: Deserializer,
    >(deserializer: &mut D) -> Result<Point, D::Error> {
        enum Field {
            x,
            y,
        }

        struct FieldVisitor;

        impl Visitor for FieldVisitor {
            type Value = Field;

            fn visit_str<
                E: Error,
            >(&mut self, value: &str) -> Result<Field, E> {
                match value {
                    "x" => Ok(Field::x),
                    "y" => Ok(Field::y),
                    _ => Err(Error::syntax_error()),
                }
            }
        }

        impl Deserialize for Field {
            fn deserialize<
                D: Deserializer,
            >(state: &mut D) -> Result<Field, D::Error> {
                state.visit(&mut FieldVisitor)
            }
        }

        struct Visitor;

        impl Visitor for Visitor {
            type Value = Point;

            fn visit_map<
                V: MapVisitor,
            >(&mut self, mut visitor: V) -> Result<Point, V::Error> {
                {
                    let mut x = None;
                    let mut y = None;

                    while let Some(key) = try!(visitor.visit_key()) {
                        match key {
                            Field::x => {
                                x = Some(try!(visitor.visit_value()));
                            }
                            Field::y => {
                                y = Some(try!(visitor.visit_value()));
                            }
                        }
                    }

                    let x = match x {
                        Some(x) => x,
                        None => {
                            return Err(Error::missing_field_error("x"));
                        }
                    };
                    let y = match y {
                        Some(y) => y,
                        None => {
                            return Err(Error::missing_field_error("y"));
                        }
                    };

                    Ok(Point {
                        x: x,
                        y: y,
                    })
                }
            }

            fn visit_named_map<
                V: MapVisitor,
            >(&mut self, name: &str, visitor: V) -> Result<Point, V::Error> {
                if name == "Point" {
                    self.visit_map(visitor)
                } else {
                    Err(Error::syntax_error())
                }
            }
        }

        deserializer.visit(&mut Visitor)
    }
}
```

It's a little more complicated, but once again there is
`#[derive_deserialize]`, which does all this work for you.

### Deserializer

Deserializers then follow the same pattern as serializers. The one difference
is that we need to provide a special hook for `Option<T>` types so formats like
JSON can treat `null` types as options.

```rust
pub trait Deserializer {
    type Error: Error;

    fn visit<
        V: Visitor,
    >(&mut self, visitor: &mut V) -> Result<V::Value, Self::Error>;

    /// The `visit_option` method allows a `Deserialize` type to inform the
    /// `Deserializer` that it's expecting an optional value. This allows
    /// deserializers that encode an optional value as a nullable value to
    /// convert the null value into a `None`, and a regular value as
    /// `Some(value)`.
    #[inline]
    fn visit_option<
        V: Visitor,
    >(&mut self, visitor: &mut V) -> Result<V::Value, Self::Error> {
        self.visit(visitor)
    }
}
```

## Performance

So how does it perform? Here's the serialization benchmarks, with yet another
ordering. This time sorted by the performance:

| language | library                   | format                 | serialization (MB/s) |
| -------- | ------------------------- | ---------------------- | -------------------- |
| Rust     | capnproto-rust            | Cap'n Proto (unpacked) | 4226                 |
| Go       | go-capnproto              | Cap'n Proto            | 3824.20              |
| Rust     | bincode                   | Binary                 | 1020                 |
| Rust     | capnproto-rust            | Cap'n Proto (packed)   | 672                  |
| Go       | gogoprotobuf              | Protocol Buffers       | 596.78               |
| Rust     | rust-msgpack              | MessagePack            | 397                  |
| **Rust** | **serde2::json** (&[u8])  | **JSON**               | **373**              |
| Rust     | rust-protobuf             | Protocol Buffers       | 357                  |
| C++      | rapidjson                 | JSON                   | 316                  |
| **Rust** | **serde2::json** (Custom) | **JSON**               | **306**              |
| **Rust** | **serde2::json** (Vec)    | **JSON**               | **288**              |
| Rust     | serde::json (Custom)      | JSON                   | 244                  |
| Rust     | serde::json (&[u8])       | JSON                   | 222                  |
| Go       | goprotobuf                | Protocol Buffers       | 214.68               |
| Rust     | serde::json (Vec)         | JSON                   | 149                  |
| Go       | ffjson                    | JSON                   | 147.37               |
| Rust     | serialize::json           | JSON                   | 183                  |
| Go       | encoding/json             | JSON                   | 80.49                |

I think it's fair to say that on at least this benchmark we've hit our
performance numbers. Writing to a preallocated buffer with `BufWriter` is 18%
*faster* than [rapidjson](https://github.com/miloyip/rapidjson) (although to be
fair they are allocating). Our `Vec<u8>` writer comes in 12% slower. What's
interesting is this custom Writer. It turns out LLVM is still having trouble
lowering our generic `Vec::push_all` into a `memcpy`. This Writer variant
however is able to get us to rapidjson's level:

```rust
fn push_all_bytes(dst: &mut Vec<u8>, src: &[u8]) {
    let dst_len = dst.len();
    let src_len = src.len();

    dst.reserve(src_len);

    unsafe {
        // we would have failed if `reserve` overflowed.
        dst.set_len(dst_len + src_len);

        ::std::ptr::copy_nonoverlapping_memory(
            dst.as_mut_ptr().offset(dst_len as isize),
            src.as_ptr(),
            src_len);
    }
}

struct MyMemWriter1 {
    buf: Vec<u8>,
}

impl Writer for MyMemWriter1 {
    #[inline]
    fn write_all(&mut self, buf: &[u8]) -> IoResult<()> {
        push_all_bytes(&mut self.buf, buf);
        Ok(())
    }
}
```

Deserialization we do much better than serde because we aren't passing around
all those tags, but we have a ways to catch up to rapidjson. Still, being just 37%
slower than the fastest JSON deserializer makes me feel pretty proud.

| language | library          | format                  | deserialization (MB/s) |
| -------- | ---------------- | ----------------------- | ---------------------- |
| Rust     | capnproto-rust   | Cap'n Proto (unpacked)  | 2123                   |
| Go       | go-capnproto     | Cap'n Proto (zero copy) | 1407.95                |
| Go       | go-capnproto     | Cap'n Proto             | 711.77                 |
| Rust     | capnproto-rust   | Cap'n Proto (packed)    | 529                    |
| Go       | gogoprotobuf     | Protocol Buffers        | 272.68                 |
| C++      | rapidjson        | JSON (sax)              | 189                    |
| C++      | rapidjson        | JSON (dom)              | 162                    |
| Rust     | bincode          | Binary                  | 142                    |
| Rust     | rust-protobuf    | Protocol Buffers        | 141                    |
| Rust     | rust-msgpack     | MessagePack             | 138                    |
| **Rust** | **serde2::json** | **JSON**                | **122**                |
| Go       | ffjson           | JSON                    | 95.06                  |
| Go       | goprotobuf       | Protocol Buffers        | 79.78                  |
| Rust     | serde::json      | JSON                    | 67                     |
| Rust     | serialize::json  | JSON                    | 25                     |
| Go       | encoding/json    | JSON                    | 22.79                  |

## Conclusion

What a long trip it's been! I hope you enjoyed it. While there are still
a few things left to port over from serde1 to serde2 (like the JSON pretty
printer), and some things probably should be renamed, I'm happy with the design
so I think it's in a place where people can start using it now. Please let me
know how it goes!
