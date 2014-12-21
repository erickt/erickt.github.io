---
layout: post
title: "Rewriting Rust Serialization, Part 3.1: Another Performance Digression"
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

Overall `serde`'s approach for serialization works out pretty well. One thing I
forgot to include in the last post was that I also have two benchmarks that are
not using `serde`, but are just safely reading and writing values.  Assuming I
haven't missed anything, they should be the upper limit in performance we can
get out of any serialization framework: Here's
[serialization](https://github.com/erickt/rust-serde/blob/master/benches/bench_log.rs#L1021):

| language | library                        | serialization (MB/s) |
| -------- | -----------------------------  | -------------------- |
| **rust** | **max without string escapes** | **353**              |
| c++      | rapidjson                      | 304                  |
| **rust** | **max with string escape**     | **234**              |
| rust     | serde::json                    | 201                  |
| rust     | serialize::json                | 147                  |
| go       | ffjson                         | 147                  |

So beyond optimizing string escaping, `serde::json` is only 14% slower than the
zero-cost version and 34% slower than `rapidjson`.

[Deserialization](https://github.com/erickt/rust-serde/blob/master/benches/bench_log.rs#L1613),
on the other hand, still has a ways to go:

| language | library                         | deserialization (MB/s) |
| -------- | ------------------------------  | ---------------------- |
| rust     | rapidjson (SAX)                 | 189                    |
| c++      | rapidjson (DOM)                 | 162                    |
| **rust** | **max with Iterator&lt;u8&gt;** | **152**                |
| go       | ffjson                          | 95                     |
| **rust** | **max with Reader**             | **78**                 |
| rust     | serde::json                     | 73                     |
| rust     | serialize::json                 | 24                     |

There are a couple interesting things here:

First, `serde::json` is built upon consuming from an `Iterator<u8>`, so we're
48% slower than our theoretical max, and 58% slower than `rapidjson`. It looks
like tagged tokens, while faster than the closures in `libserialize`, are still
pretty expensive.

Second, `ffjson` is beating us and they compile dramatically faster too. The
[goser](https://github.com/cloudflare/goser) test suite takes about 0.54
seconds to compile, whereas mine takes about 30 seconds at `--opt-level=3`
(!!). Rust itself is only taking 1.5 seconds, the rest is spent in LLVM. With
no optimization, it compiles "only" in 5.6 seconds, and is 96% slower.

Third, `Reader` is a surprisingly expensive trait when dealing with a format
like JSON that need to read a byte at a time. It turns out we're not
[generating great code](https://github.com/rust-lang/rust/issues/19864) for
types with padding. aatch has been working on fixing this though.

---

Since I wrote that last section, I did a little more experimentation to try to
figure out why our serialization upper bound is 23% slower than rapidjson. And,
well, maybe I found it?

|                                | serialization (MB/s) |
| ------------------------------ | -------------------- |
| serde::json with a MyMemWriter | 346                  |
| serde::json with a Vec<u8>     | 247                  |

All I did with `MyMemWriter` is copy the `Vec::<u8>` implementation of `Writer`
into the local codebase:

```rust
struct MyMemWriter0 {
    buf: Vec<u8>,
}

impl MyMemWriter0 {
    pub fn with_capacity(cap: uint) -> MyMemWriter0 {
        MyMemWriter0 {
            buf: Vec::with_capacity(cap)
        }
    }
}

impl Writer for MyMemWriter0 {
    #[inline]
    fn write(&mut self, buf: &[u8]) -> io::IoResult<()> {
        self.buf.push_all(buf);
        Ok(())
    }
}

#[bench]
fn bench_serializer_my_mem_writer0(b: &mut Bencher) {
    let log = Log::new();
    let json = json::to_vec(&log);
    b.bytes = json.len() as u64;

    let mut wr = MyMemWriter0::with_capacity(1024);

    b.iter(|| {
        wr.buf.clear();

        let mut serializer = json::Serializer::new(wr.by_ref());
        log.serialize(&mut serializer).unwrap();
        let _json = serializer.unwrap();
    });
}

#[bench]
fn bench_serializer_vec(b: &mut Bencher) {
    let log = Log::new();
    let json = json::to_vec(&log);
    b.bytes = json.len() as u64;

    let mut wr = Vec::with_capacity(1024);

    b.iter(|| {
        wr.clear();

        let mut serializer = json::Serializer::new(wr.by_ref());
        log.serialize(&mut serializer).unwrap();
        let _json = serializer.unwrap();
    });
}
```

Somehow it's not enough to just mark `Vec::write` as
`#[inline]`, having it in the same file gave LLVM enough information to
optimize it's overhead away. Even using `#[inline(always)]` on `Vec::write` and
`Vec::push_all` isn't able to get the same increase, so I'm not sure how to
replicate this in the general case.

Also interesting is `bench_serializer_slice`, which uses `BufWriter`.

|                                | serialization (MB/s) |
| ------------------------------ | -------------------- |
| serde::json with a BufWriter   | 342                  |


```rust
#[bench]
fn bench_serializer_slice(b: &mut Bencher) {
    let log = Log::new();
    let json = json::to_vec(&log);
    b.bytes = json.len() as u64;

    let mut buf = [0, .. 1024];

    b.iter(|| {
        for item in buf.iter_mut(){ *item = 0; }
        let mut wr = std::io::BufWriter::new(&mut buf);

        let mut serializer = json::Serializer::new(wr.by_ref());
        log.serialize(&mut serializer).unwrap();
        let _json = serializer.unwrap();
    });
}
```

---

Another digression. Since I wrote the above, aatch has put out some PRs that
should help speed up enums.
[19898](https://github.com/rust-lang/rust/pull/19898) and
[#20060](https://github.com/rust-lang/rust/pull/20060) and was able to optimize
the padding out of enums and fix an issue with returns generating bad code. In
my [bug from earlier](https://github.com/rust-lang/rust/issues/19864)
his patches were able to speed up my benchmark returning an
`Result<(), IoError>` from running at 40MB/s to 88MB/s. However, if we're able
to reduce `IoError` down to a word, we get the performance up to 730MB/s! We
also might get enum compression, so a type like `Result<(), IoError>` then
would speed up to 1200MB/s! I think going in this direction is going to really
help speed things up.
