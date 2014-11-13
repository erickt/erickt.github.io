---
layout: post
title: "Rewriting Rust Serialization, Part 2.2: More Benchmarks"
date: 2014-11-13 09:07:36 -0800
comments: true
categories: [rust, serialization]
---

Back to the benchmarks! I got some great comments on
[reddit](https://www.reddit.com/r/rust/comments/2lzc9n/rust_serialization_part_21_now_with_more/),
So I wanted to do another post to update my numbers. Here's what I changed:

* I wasn't consistent on whether or not the serialization benchmarks included 
  Some tests are including the allocation of a buffer to write into. I've
	changed it so most are reusing one, which speeds everything up (especially
  capnproto-rust!). This does depend on
  [#18885](https://github.com/rust-lang/rust/pull/18885) landing though.
* I've added [bincode](https://github.com/TyOverby/bincode), which serializes
  values as raw bytes. Quite speedy too! Not nearly as fast as Cap'n Proto though.
* I've changed `C++` and `Rust` JSON tests to serialize enums as uints.
* I added the time it takes to create the populate the structures. I'm betting
	the reason the Rust numbers are so high is that we're allocating strings. Not
  sure if the other languages are able to avoid that allocation.

---

JSON:

| language | library         | population (ns) | serialization (MB/s) | deserialization (MB/s) |
| -------- | --------------- | --------------- | -------------------- | ---------------------- |
| Rust     | serialize::json | 1127            | 117                  | 26                     |
| C++      | rapidjson (dom) | 546             | 281                  | 144                    |
| C++      | rapidjson (dom) | 546             | 281                  | 181                    |
| Go       | encoding/json   | 343             | 63.99                | 22.46                  |
| Go       | ffjson          | 343             | 144.60               | (not supported)        |

---

Cap'n Proto:

| language | library                   | population (ns) | serialization (MB/s) | deserialization (MB/s) |
| -------- | ------------------------- | --------------- | -------------------- | ---------------------- |
| Rust     | capnproto-rust (unpacked) | 325             | 4977                 | 2251                   |
| Rust     | capnproto-rust (packed)   | 325             | 398                  | 246                    |
| Go       | go-capnproto              | 2368            | 2226.71              | 450                    |
| Go       | go-capnproto (zero copy)  | 2368            | 2226.71              | 1393.3                 |

---

Protocol Buffers:

| language | library       | population (ns) | serialization (MB/s) | deserialization (MB/s) |
| -------- | ------------  | --------------- | -------------------- | ---------------------- |
| Rust     | rust-protobuf | 1041            | 370                  | 118                    |
| Go       | goprotobuf    | 1133            | 138.27               | 91.18                  |
| Go       | gogoprotobuf  | 343             | 472.69               | 295.33                 |

---

Misc:

| language | library      | population (ns) | serialization (MB/s) | deserialization (MB/s) |
| -------- | ------------ | --------------- | -------------------- | ---------------------- |
| Rust     | rust-msgpack | 1143            | 454                  | 144                    |
| Rust     | bincode      | 1143            | 1149                 | 82                     |

Anyone want to add more C/Go/Rust/Java/etc benchmarks?
