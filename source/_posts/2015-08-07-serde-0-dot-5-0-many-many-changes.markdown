---
layout: post
title: "Serde 0.5.0 - Many Many Changes"
date: 2015-08-07 08:14:22 -0700
comments: true
categories: [rust, serialization, serde]
---

Hello all you beautiful and talented people! I'm pleased to announce
[serde](https://github.com/serde-rs/serde) 0.5.0. We're bumping the major
(unstable) version number here because there have been a huge amount of
breaking changes in the API. This has been done to better support serialization
formats like [bincode](https://github.com/TyOverby/bincode), which relies on
the `Serialize`e to hint to the `Serializer` how to parse the next bytes.
This will enable [Servo](https://github.com/servo/servo/pull/6583) to use
bincode for it's IPC protocol.

Here are the major changes:

* `serde::json` was factored out into it's own separate crate
  [serde\_json](https://crates.io/crates/serde_json)
	[#114](https://github.com/serde-rs/serde/pull/114).
* Added serialization and deserialization type hints.
* Renamed many functions to change `visit_named_{map,seq}` to
	`visit_struct` and `visit_tuple_struct`
	[#114](https://github.com/serde-rs/serde/pull/114)
  [#120](https://github.com/serde-rs/serde/pull/120).
* Added hooks to allow serializers to serialize newtype tuple structs without a
  wrapper type [#121](https://github.com/serde-rs/serde/pull/121).
* Remove `_error` from `de::Error`
  [#129](https://github.com/serde-rs/serde/pull/129).
* Rewrote json parser to not consume the whole stream
  [#127](https://github.com/serde-rs/serde/pull/127).
* Fixed `serde_macros` for generating fully generic code
  [#117](https://github.com/serde-rs/serde/pull/117).


Thank you to everyone that's helped with this release:

* Craig Brandenburg
* Hugo Duncan
* Jarred Nicholis
* Oliver Schneider
* Patrick Walton
* Sebastian Thiel
* Skylar Lipthay
* Thomas Bahn
* dswd

Benchmarks
==========

It's been a bit since we last did some
[benchmarks](https://erickt.github.io/blog/2015/02/16/rewriting-rust-serialization-there-can-be-only-one-serde/),
so here are the latest numbers with these compilers:

* rustc: 1.4.0-nightly (1181679c8 2015-08-07)
* go: version go1.4.2 darwin/amd64
* clang: Apple LLVM version 6.1.0 (clang-602.0.53) (based on LLVM 3.6.0svn)

[bincode](https://github.com/TyOverby/bincode)'s serde support makes it's first
appearance, which starts out roughly 1/3 slower at serialization, but about the
same speed at deserialization. I haven't done much optimization, so there's
probably a lot of low hanging fruit.

[serde_json](https://crates.io/crates/serde_json) saw a good amount of
improvement, mainly from some compiler optimizations in the 1.4 nightly. The
deserializer is slightly slower due to the parser rewrite.

[capnproto-rust](https://github.com/dwrensha/capnproto-rust)'s unpacked format
shows a surprisingly large large serialization improvement, with a 10x
improvement from 4GB/s to 15GB/s. Good job dwrensha!  Deserialization is half
as slow as before though. Perhaps I have a bug in my code?

I've changed the Rust MessagePack implementation to
[rmp](https://github.com/3Hren/msgpack-rust), which has a wee bit faster
serializer, but deserialization was about the same.

I've also updated the numbers for Go and C++, but those numbers stayed roughly
the same.

Serialization:

| language | library             | format                     | serialization (MB/s) |
| -------- | ---------------     | -------------------------- | -------------------- |
| **Rust** | **capnproto-rust**  | **Cap'n Proto (unpacked)** | ~~4349~~ **15448**   |
| Go       | go-capnproto        | Cap'n Proto                | 3877                 |
| **Rust** | **bincode**         | **Raw**                    | ~~1020~~ **3278**    |
| **Rust** | **bincode (serde)** | **Raw**                    | **2143**             |
| **Rust** | **capnproto-rust**  | **Cap'n Proto (packed)**   | ~~583~~ **656**      |
| Go       | gogoprotobuf        | Protocol Buffers           | ~~596~~ 627          |
| **Rust** | **rmp**             | **MessagePack**            | ~~397~~ **427**      |
| **Rust** | **rust-protobuf**   | **Protocol Buffers**       | ~~357~~ **373**      |
| **Rust** | **serde::json**     | **JSON**                   | ~~288~~ **337**      |
| C++      | rapidjson           | JSON                       | 307                  |
| Go       | goprotobuf          | Protocol Buffers           | ~~214~~ 226          |
| **Rust** | **serialize::json** | **JSON**                   | ~~147~~ **212**      |
| Go       | ffjson              | JSON                       | 147                  |
| Go       | encoding/json       | JSON                       | 85                   |

Deserialization:

| language | library             | format                     | deserialization (MB/s) |
| -------- | ----------------    | -----------------------    | ---------------------- |
| **Rust** | **capnproto-rust**  | **Cap'n Proto (unpacked)** | ~~2185~~ **1306**      |
| Go       | go-capnproto        | Cap'n Proto (zero copy)    | 1407                   |
| Go       | go-capnproto        | Cap'n Proto                | 711                    |
| **Rust** | **capnproto-rust**  | **Cap'n Proto (packed)**   | ~~351~~ **464**        |
| **Rust** | **bincode (serde)** | **Raw**                    | **310**                |
| **Rust** | **bincode**         | **Raw**                    | ~~142~~ **291**        |
| Go       | gogoprotobuf        | Protocol Buffers           | 270                    |
| C++      | rapidjson           | JSON (sax)                 | 182                    |
| C++      | rapidjson           | JSON (dom)                 | 155                    |
| **Rust** | **rust-protobuf**   | **Protocol Buffers**       | **143**                |
| **Rust** | **rmp**             | **MessagePack**            | ~~138~~ **128**        |
| **Rust** | **serde::json**     | **JSON**                   | ~~140~~ **122**        |
| Go       | ffjson              | JSON                       | 95                     |
| Go       | goprotobuf          | Protocol Buffers           | 81                     |
| Go       | encoding/json       | JSON                       | 23                     |
| Rust     | serialize::json     | JSON                       | 23                     |
