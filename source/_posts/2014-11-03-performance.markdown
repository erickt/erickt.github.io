---
layout: post
title: "Rewriting Rust Serialization, Part 2: Performance"
date: 2014-11-03 07:38:38 -0700
comments: true
categories: [rust, serialization]
---

As I said in the [last post](http://erickt.github.io/blog/2014/10/28/serialization/),
Rust's `serialize` library, specifically `serialize::json` is pretty slow.
Back when I started this project a number of months ago, I wanted to benchmark
to see how we compared to some other languages. There are a bunch of JSON
benchmarks, but the one I chose was Cloudflare's Go language.
[Goser](https://github.com/cloudflare/goser), mainly because it was using a
complex real world log structure, and they did the hard work of implementing
benchmarks for [encoding/json](http://golang.org/pkg/encoding/json),
[goprotobuf](http://code.google.com/p/goprotobuf/),
[gogoprotobuf](http://code.google.com/p/gogoprotobuf/), and
[go-capnproto](https://github.com/glycerine/go-capnproto). I also included the
Go [ffjson](https://github.com/pquerna/ffjson) and C++
[rapidjson](https://github.com/erickt/rapidjson/blob/master/log.cc), which
both claim to be the fastest JSON libraries for those languages. Here are the
results I got:

| language | library             | format           | serialization (MB/s) | deserialization (MB/s) |
| -------- | ------------------- | ---------------- | -------------------- | ---------------------- |
| C++      | rapidjson           | JSON             | 294                  | 164 (DOM) / 192 (SAX)  |
| Go       | encoding/json       | JSON             | 71.47                | 25.09                  |
| Go       | ffjson              | JSON             | 156.67               | (not supported)        |
| Go       | goprotobuf          | Protocol Buffers | 148.78               | 99.57                  |
| Go       | gogoprotobuf        | Protocol Buffers | 519.48               | 319.40                 |
| Go       | go-capnproto        | Cap'n Proto      | 3419.54              | 665.35                 |
| Rust     | serialize::json     | JSON             | 40-ish               | 10-ish                 |

Notes:

* `rapidjson` supports both DOM-style and SAX-style deserializing. DOM-style
  means deserializing into a generic object, then from there into the final
  object, SAX-style means a callback approach where a callback handler is
  called for each JSON token.
* Go's `encoding/json` uses reflection to serialize arbitrary values. `ffjson`
  uses code generation to get it's serialization sped, but it doesn't implement
  deserialization.
* both `goprotobuf` and `gogoprotobuf` use code generation, but gogoprotobuf
  uses Protocol Buffer's extension support to do cheaper serialization.
* Cap'n Proto doesn't really do serialization, but lays the serialized data out
  just like it is in memory so it has nearly zero serialization speed.
* The Rust numbers are from a couple months ago and I couldn't track down
  the exact numbers.

So. Yikes. Not only are we no where near `rapidjson`, we were being soundly
beaten by Go's reflection-based framework `encoding/json`.  Even worse, our
compile time was at least 10 times theirs. So, not pretty at all.

But that was a couple months ago. Between then and now, Patrick Walton, Luqman
Aden, myself, and probably lots others found and fixed a number of bugs across
`serialize::json`, `std::io`, generic function calls, and more. All this work
got us to more than double our performance:

| language | library           | format               | serialization (MB/s)   | deserialization (MB/s) |
| -------- | ----------------- | -------------------- | ---------------------- | ---------------------- |
| Rust     | serialize::json   | JSON                 | 117                    | 25                     |

We're (kind of) beating Go! At least the builtin reflection-based solution.
Better, but not great. I think our challenge is those dang closures. While LLVM
can optimize simple closures, it seems to have a lot of trouble with all these
recursive closure calls. While having finished unboxed closures might finally
let us break through this performance bottleneck, it's not guaranteed.

All in all, this, and the representational problems from
[post 1](http://erickt.github.io/blog/2014/10/28/serialization/) make it pretty
obvious we got some fundamental issues here and we need to use an alternative
solution. Next post I'll start getting into the details of the design of
[serde](https://github.com/erickt/rust-serde).
