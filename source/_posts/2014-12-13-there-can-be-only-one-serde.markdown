---
layout: post
title: "Rewriting Rust Serialization, Part 4: There can be only one"
date: 2014-12-13 20:35:08 -0800
comments: true
categories: [rust, serialization]
---

Wow, home stretch! Here's the rest of the series if you want to catch up:
[part 1](http://erickt.github.io/blog/2014/10/28/serialization/),
[part 2](http://erickt.github.io/blog/2014/11/03/performance/),
[part 2.1](http://erickt.github.io/blog/2014/11/03/performance/),
[part 2.2](http://erickt.github.io/blog/2014/11/03/performance/), and
[part 3](http://erickt.github.io/blog/2014/12/13/rewriting-rust-serialization/).

We talked about serde the last time. Overall it's approach for serialization
works out really well. I forgot to mention that as part of my benchmark suite I
have two tests for benchmarking
[serialization](https://github.com/erickt/rust-serde/blob/master/benches/bench_log.rs#L1021)
and
[deserialization](https://github.com/erickt/rust-serde/blob/master/benches/bench_log.rs#L1613)
with no abstraction overhead, just safely reading and writing values from and
to a `String`. Assuming I didn't make any mistakes, they should theoretically
be the upper limit in performance we can get out of any serialization
framework: Here's serialization:

|                         | serialization (MB/s) |
| manual without escaping | 353                  |
| manual with escape      | 234                  |
| serde::json             | 201                  |
| serialize::json         | 147                  |

So beyond optimizing string escaping, we are in spitting distance of having
zero-cost serialization, which is pretty awesome. Deserializing has a bit more
of a ways to go:

|                      | deserialization (MB/s) |
| manual with Iterator | 152                    |
| manual with Reader   | 78                     |
| serde::json          | 73                     |
| serialize::json      | 24                     |

`serde::json` is already built upon `Iterator<u8>`, so we're paying about a 50%
tax for our abstraction. I included `Reader` here to just show how expensive
can be to use rust's `Reader` library when dealing with byte-oriented protocols
like JSON. It also suggests some optimization opportunites for the `&[u8]`
`Reader` implementation.

The other thing that's interesting to note is that
[rapidjson](https://github.com/miloyip/rapidjson) serializing at 304 MB/s and
deserializing at 188 MB/s is still somehow faster than the best we can do. I
suspect there must be something holding us back. I don't have any proof, but I
wonder if
[zero-on-drop](http://discuss.rust-lang.org/t/the-sad-state-of-zero-on-drop/944)
is blocking some optimizations that would really get us over this performance
hurdle.

