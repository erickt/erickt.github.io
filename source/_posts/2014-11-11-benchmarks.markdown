---
layout: post
title: "Rewriting Rust Serialization, Part 2.1: Benchmarks"
date: 2014-11-11 08:11:34 -0800
comments: true
categories: [rust, serialization]
---

After [part 2](http://erickt.github.io/blog/2014/11/03/performance/) I received
a couple requests to add in a couple other rust serialization libraries. So one
thing led to another, and now I've got a benchmark suite I'm calling
[rust-serialization-benchmarks](https://github.com/erickt/rust-serialization-benchmarks).
Really creative name, eh? This includes all the other benchmarks I referred to
previously, as well as [capnproto](https://github.com/dwrensha/capnproto-rust),
[msgpack](https://github.com/mneumann/rust-msgpack), and
[protobuf](https://github.com/stepancheg/rust-protobuf).

| language | library             | format                  | serialization (MB/s) | deserialization (MB/s)   |
| -------- | ------------------- | ----------------------- | -------------------- | ------------------------ |
| C++      | rapidjson           | JSON (dom)              | 233                  | 102                      |
| C++      | rapidjson           | JSON (sax)              | 233                  | 124                      |
| Go       | encoding/json       | JSON                    | 54.93                | 16.72                    |
| Go       | ffjson              | JSON                    | 126.40               | (not supported)          |
| Go       | goprotobuf          | Protocol Buffers        | 138.27               | 91.18                    |
| Go       | gogoprotobuf        | Protocol Buffers        | 472.69               | 295.33                   |
| Go       | go-capnproto        | Cap'n Proto             | 2226.71              | 450                      |
| Go       | go-capnproto        | Cap'n Proto (zero copy) | 2226.71              | 1393.3                   |
| Rust     | serialize::json     | JSON                    | 89                   | 18                       |
| Rust     | rust-msgpack        | MessagePack             | 160                  | 52                       |
| Rust     | rust-protobuf       | Protocol Buffers        | 177                  | 70                       |
| Rust     | capnproto-rust      | Cap'n Proto (unpacked)  | 1729                 | 1276                     |
| Rust     | capnproto-rust      | Cap'n Proto (packed)    | 398                  | 246                      |

I upgraded to OS X Yosemite, so I think that brought these numbers down overall
from the last post.
