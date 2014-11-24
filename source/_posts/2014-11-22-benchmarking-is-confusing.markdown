---
layout: post
title: "Benchmarking is Confusing in Low Level Rust"
date: 2014-11-22 12:10:29 -0800
comments: true
categories: [benchmarking, rust]
---

Low level benchmarking is confusing and non-intuitive.

The end.

---

Or not. Whatever. So I'm trying to get my
implement-`Reader`-and-`Writer`-for-`&[u8]` type PR
[#18980](https://github.com/rust-lang/rust/pull/18980) landed. But
[Steven Fackler](https://github.com/rust-lang/rust/pull/18980#issuecomment-63925495)
obnixously and correctly pointed out that this won't play that nicely with the
new `Reader` and `Writer` implementation for `Vec<u8>`. Grumble grumble. And then 
[Alex Crichton](https://github.com/rust-lang/rust/pull/18980#issuecomment-63927659)
had the gall to mention that a `Writer` for `mut &[u8]` also probably won't be
that common either. Sure, he's write and all, but but I got it working without
needing an index! That means that the `&mut [u8]` `Writer` only needs 2
pointers instead of `BufWriter`'s three, so it just has to be faster! Well,
doesn't it?

Stupid benchmarks.

I got to say it's pretty addicting writing micro-benchmarks. It's a lot of fun
seeing how sensitive low-level code can be to just the smallest of tweaks. It's
also really annoying when you write something you think is pretty neat, then
you find it's chock-full of false dependencies between cache lines, or other
mean things CPUs like to impose on poor programmers.

Anyway, to start lets look at what should be the fastest way to write to a
buffer. Completely unsafely with no checks.

```rust
unsafe fn do_copy_nonoverlapping_memory(
  mut dst: *mut u8,
  src: *const u8,
  len: uint,
  batches: uint
) {
    for _ in range(0, batches) {
        ptr::copy_nonoverlapping_memory(dst, src, len);
        dst = dst.offset(len as int);
    }
}

#[test]
fn test_copy_nonoverlapping_memory() {
    let dst = &mut [0_u8, .. BATCHES * SRC_LEN];
    let src = &[1, .. SRC_LEN];

    unsafe {
        do_copy_nonoverlapping_memory(
            dst.as_mut_ptr(),
            src.as_ptr(),
            src.len(),
            BATCHES
        );
    }
    assert!(dst.iter().all(|c| *c == 1));
}
```

With `SRC_LEN=4` and `BATCHES=128`, we get this. For fun I added the new
`libtest` from [#19233](https://github.com/rust-lang/rust/pull/19233) that will
hopefully land soon. I also added also ran variations that explicitly inlined
and not inlined the inner function:

```
test bench_copy_nonoverlapping_memory               ... bench: 50 |   [-***#**------]                        | 200:        72 ns/iter (+/- 45)
test bench_copy_nonoverlapping_memory_inline_always ... bench: 50 |       [---***#****************----------]| 100:        65 ns/iter (+/- 39)
test bench_copy_nonoverlapping_memory_inline_never  ... bench: 500 |      [------********#*********-------] | 1000:       747 ns/iter (+/- 393)
```

So overall it does quite well. Now lets compare with the code I wrote:

```rust
impl<'a> Writer for &'a mut [u8] {
    #[inline]
    fn write(&mut self, src: &[u8]) -> IoResult<()> {
        if self.is_empty() { return Err(io::standard_error(io::EndOfFile)); }

        let dst_len = self.len();
        let src_len = src.len();

        let write_len = min(dst_len, src_len);

        slice::bytes::copy_memory(*self, src.slice_to(write_len));

        unsafe {
            *self = mem::transmute(raw::Slice {
                data: self.as_ptr().offset(write_len as int),
                len: dst_len - write_len,
            });
        }

        if src_len > dst_len {
            Err(io::standard_error(io::ShortWrite(write_len)))
        } else {
            Ok(())
        }
    }
}
```

And. Well. It didn't do that well.

```
test writer::bench_slice_writer                             ... bench: 500 |   [------**#**--]                      | 2000:       920 ns/iter (+/- 448)
test writer::bench_slice_writer_inline_always               ... bench: 600 | [-**#*****---]                         | 2000:       711 ns/iter (+/- 405)
test writer::bench_slice_writer_inline_never                ... bench: 600 |   [-***#******---]                     | 2000:       838 ns/iter (+/- 474)
```

Wow. That's pretty bad compared to the ideal. 

Crud. So not only did I add an implementation that's probably going to not work
with `write!` and now it turns out the performance is pretty terrible. Inlining
isn't helping like it did in the unsafe case. So how's
[std::io::BufWriter](https://github.com/rust-lang/rust/blob/master/src/libstd/io/mem.rs#L219)
compare?

```rust
pub struct BufWriter<'a> {
    buf: &'a mut [u8],
    pos: uint
}

impl<'a> Writer for BufWriter<'a> {
    #[inline]
    fn write(&mut self, buf: &[u8]) -> IoResult<()> {
        // return an error if the entire write does not fit in the buffer
        let cap = if self.pos >= self.buf.len() { 0 } else { self.buf.len() - self.pos };
        if buf.len() > cap {
            return Err(IoError {
                kind: io::OtherIoError,
                desc: "Trying to write past end of buffer",
                detail: None
            })
        }

        slice::bytes::copy_memory(self.buf[mut self.pos..], buf);
        self.pos += buf.len();
        Ok(())
    }
}
```

Here's how it does:

```
test writer::bench_std_buf_writer                           ... bench: 50 |     [------**************#******-------] | 100:        79 ns/iter (+/- 40)
test writer::bench_std_buf_writer_inline_always             ... bench: 50 |    [-----************#*********-------]  | 100:        75 ns/iter (+/- 40)
test writer::bench_std_buf_writer_inline_never              ... bench: 600 |   [#****-----]                         | 2000:       705 ns/iter (+/- 337)
```

That's just cruel. The optimization gods obviously hate me. So I started
playing with a lot of
[variations](https://github.com/erickt/rust-serialization-benchmarks/blob/master/rust/src/writer.rs)
(it's my yeah yeah it's my serialization benchmark suite, I'm planning on
making it more general purpose. Besides it's my suite and I can do whatever I
want with it, so there):

* (BufWriter0): Turning this `Writer` into a struct wrapper shouldn't do anything, and it
  didn't.
* (BufWriter1): There's error handling, does removing it help? Nope!
* (BufWriter5): There's an implied branch in `let write_len = min(dst_len, src_len)`. We can
  turn that into the branch-predictor-friendly:

```rust
let x = (dst_len < buf_len) as uint;
let write_len = dst_len * x + src_len * (1 - x);
```

Doesn't matter, still performs the same.

* (BufWriter2): Fine then, optimization gods! Lets remove the branch altogether and just
	always advance the slice `src.len()` bytes! Damn the safety! That, of course,
  works. I can hear them giggle.
* (BufWriter3): Maybe, just maybe there's something weird going on with
	inlining across crates? Lets copy `std::io::BufWriter` and make sure that
	it's still nearly optimal. It still is.
* (BufWriter6): Technically the `min(dst_len, src_len)` is a bounds check, so
  we could switch from the bounds checked `std.slice::bytes::copy_memory` to
	the unsafe `std::ptr::copy_nonoverlapping_memory`, but that also doesn't
	help.
* (BufWriter7): Might as well and apply the last trick to `std::io::BufWriter`,
	and it does shave a couple nanoseconds off. It might be worth pushing it
	upstream:

```
test writer::bench_buf_writer_7                             ... bench: 50 |   [-*#********--------------------------]| 100:        55 ns/iter (+/- 44)
test writer::bench_buf_writer_7_inline_always               ... bench: 50 |     [---------********#*********-------] | 100:        76 ns/iter (+/- 39)
test writer::bench_buf_writer_7_inline_never                ... bench: 600 |   [-***#****----]                      | 2000:       828 ns/iter (+/- 417)
```

* (BufWriter4): While I'm using one less `uint` than `std::io::BufWriter`, I'm
	doing two writes to advance my slice, one to advance the pointer, and one to
	shrink the length. `std::io::BufWriter` only has to advance it's `pos` index.
	But in this case if instead of treating the slice as a `(ptr, length)`, we
	can convert it into a `(start_ptr, end_ptr)`, where `start_ptr=ptr`, and
	`end_ptr=ptr+length`. This works! Ish:

```
test writer::bench_buf_writer_4                             ... bench: 80 |   [--******#*******-----]                | 200:       109 ns/iter (+/- 59)
test writer::bench_buf_writer_4_inline_always               ... bench: 100 |     [------***#******---]               | 200:       133 ns/iter (+/- 44)
test writer::bench_buf_writer_4_inline_never                ... bench: 500 |      [-----***********#***********----]| 1000:       778 ns/iter (+/- 426)
```

I know when I'm defeated. Oh well. I guess I can at least update
`std::io::BufWriter` to support the new error handling approach:

```rust
impl<'a> MyWriter for BufWriter10<'a> {
    #[inline]
    fn my_write(&mut self, src: &[u8]) -> IoResult<()> {
        let dst_len = self.dst.len();

        if self.pos == dst_len { return Err(io::standard_error(io::EndOfFile)); }

        let src_len = src.len();
        let cap = dst_len - self.pos;

        let write_len = min(cap, src_len);

        slice::bytes::copy_memory(self.dst[mut self.pos..], src[..write_len]);

        if src_len > dst_len {
            return Err(io::standard_error(io::ShortWrite(write_len)));
        }

        self.pos += write_len;
        Ok(())
    }
}
```

How's it do?

```
test writer::bench_buf_writer_10                            ... bench: 600 | [----**#***--]                         | 2000:       841 ns/iter (+/- 413)
test writer::bench_buf_writer_10_inline_always              ... bench: 600 |  [----**#***--]                        | 2000:       872 ns/iter (+/- 387)
test writer::bench_buf_writer_10_inline_never               ... bench: 600 |   [--******#**---]                     | 2000:       960 ns/iter (+/- 486)
```

Grumble grumble. It turns out that if we tweak the `copy_memory` line to:

```rust
        slice::bytes::copy_memory(self.dst[mut self.pos..], src);
```

It shaves 674 nanoseconds off the run:

```
test writer::bench_buf_writer_10                            ... bench: 200 |[---***#************------------]        | 400:       230 ns/iter (+/- 147)
test writer::bench_buf_writer_10_inline_always              ... bench: 200 |   [-----********#*******------]         | 400:       280 ns/iter (+/- 128)
test writer::bench_buf_writer_10_inline_never               ... bench: 600 |   [--***#****----]                     | 2000:       885 ns/iter (+/- 475)
```

But still no where near where we need to be. That suggests though that always
cutting down the `src`, which triggers another bounds check has some measurable
impact. So maybe I should only shrink the `src` slice when we know it needs to
be shrunk?

```rust
impl<'a> MyWriter for BufWriter11<'a> {
    #[inline]
    fn my_write(&mut self, src: &[u8]) -> IoResult<()> {
        let dst = self.dst[mut self.pos..];
        let dst_len = dst.len();

        if dst_len == 0 {
            return Err(io::standard_error(io::EndOfFile));
        }

        let src_len = src.len();

        if dst_len >= src_len {
            unsafe {
                ptr::copy_nonoverlapping_memory(
                    dst.as_mut_ptr(),
                    src.as_ptr(),
                    src_len);
            }

        		self.pos += src_len;

            Ok(())
        } else {
            unsafe {
                ptr::copy_nonoverlapping_memory(
                    dst.as_mut_ptr(),
                    src.as_ptr(),
                    dst_len);
            }

        		self.pos += dst_len;

            Err(io::standard_error(io::ShortWrite(dst_len)))
        }
    }
}
```

Lets see how it failed this time...

```
test writer::bench_buf_writer_11                            ... bench: 60 |      [------********#*********-----]     | 100:        79 ns/iter (+/- 28)
test writer::bench_buf_writer_11_inline_always              ... bench: 60 |[-------******#*************-----------]  | 100:        72 ns/iter (+/- 35)
test writer::bench_buf_writer_11_inline_never               ... bench: 600 |  [--***#***----]                       | 2000:       835 ns/iter (+/- 423)
```

No way. That actually worked?! That's awesome! I'll carve that out into another
PR. Maybe it'll work for my original version that doesn't use a `pos`?

```rust
impl<'a> MyWriter for BufWriter12<'a> {
    #[inline]
    fn my_write(&mut self, src: &[u8]) -> IoResult<()> {
        let dst_len = self.dst.len();

        if dst_len == 0 {
            return Err(io::standard_error(io::EndOfFile));
        }

        let src_len = src.len();

        if dst_len >= src_len {
            unsafe {
                ptr::copy_nonoverlapping_memory(
                    self.dst.as_mut_ptr(),
                    src.as_ptr(),
                    src_len);

                self.dst = mem::transmute(raw::Slice {
                    data: self.dst.as_ptr().offset(src_len as int),
                    len: dst_len - src_len,
                });
            }

            Ok(())
        } else {
            unsafe {
                ptr::copy_nonoverlapping_memory(
                    self.dst.as_mut_ptr(),
                    src.as_ptr(),
                    dst_len);

                self.dst = mem::transmute(raw::Slice {
                    data: self.dst.as_ptr().offset(dst_len as int),
                    len: 0,
                });
            }

            Err(io::standard_error(io::ShortWrite(dst_len)))
        }
    }
}
```

And yep, just as fast!

```
test writer::bench_buf_writer_12                            ... bench: 50 |[**#*----------------------------------]   | 80:        51 ns/iter (+/- 26)
test writer::bench_buf_writer_12_inline_always              ... bench: 50 |  [--------**********#********------------]| 90:        69 ns/iter (+/- 36)
test writer::bench_buf_writer_12_inline_never               ... bench: 800 |  [---**#***-]                          | 2000:      1000 ns/iter (+/- 263)
```

At this point, both solutions are approximately just as fast as the unsafe
`ptr::copy_nonoverlapping_memory`! So that's awesome. Now would anyone really
care enough about the extra `uint`?  There may be a few very specialized cases
where that extra `uint` could cause a problem, but I'm not sure if it's worth
it. What do you all think?

---

I thought that was good, but since I'm already here, how's the new `Vec<u8>`
writer doing? Here's the driver:

```rust
impl Writer for Vec<u8> {
    #[inline]
    fn write(&mut self, buf: &[u8]) -> IoResult<()> {
        self.push_all(buf);
        Ok(())
    }
}

impl Writer for Vec<u8> {
    #[inline]
    fn write(&mut self, buf: &[u8]) -> IoResult<()> {
        self.push_all(buf);
        Ok(())
    }
}

#[bench]
fn bench_std_vec_writer(b: &mut test::Bencher) {
    let mut dst = Vec::with_capacity(BATCHES * SRC_LEN);
    let src = &[1, .. SRC_LEN];

    b.iter(|| {
        dst.clear();

        do_std_writes(&mut dst, src, BATCHES);
    })
}
```

And the results:

```
test writer::bench_std_vec_writer                           ... bench: 1000 | [----*****#*****--------]             | 2000:      1248 ns/iter (+/- 588)
test writer::bench_std_vec_writer_inline_always             ... bench: 900 |   [----*#***--]                        | 2000:      1125 ns/iter (+/- 282)
test writer::bench_std_vec_writer_inline_never              ... bench: 1000 |  [----***#*****--------]              | 2000:      1227 ns/iter (+/- 516)
```

Wow. That's pretty terrible. Something weird must be going on with
`Vec::push_all`. (Maybe that's what caused my serialization benchmarks to slow
1/3?). Lets skip it:

```rust
impl<'a> MyWriter for VecWriter1<'a> {
    #[inline]
    fn my_write(&mut self, src: &[u8]) -> IoResult<()> {
        let src_len = src.len();

        self.dst.reserve(src_len);

        let dst = self.dst.as_mut_slice();

        unsafe {
            // we reserved enough room in `dst` to store `src`.
            ptr::copy_nonoverlapping_memory(
                dst.as_mut_ptr(),
                src.as_ptr(),
                src_len);
        }

        Ok(())
    }
}
```

And it looks a bit better, but not perfect:

```
test writer::bench_vec_writer_1                             ... bench: 100 |         [------*********#*****--------] | 200:       160 ns/iter (+/- 68)
test writer::bench_vec_writer_1_inline_always               ... bench: 100 |     [--------****#**--]                 | 300:       182 ns/iter (+/- 79)
test writer::bench_vec_writer_1_inline_never                ... bench: 600 |   [---****#**--]                       | 2000:       952 ns/iter (+/- 399)
```

There's even less going on here than before. The only difference is that
reserve call. Commenting it out gets us back to `copy_nonoverlapping_memory`
territory:

```
test writer::bench_vec_writer_1                             ... bench: 70 | [----------------*********#******-------]| 100:        89 ns/iter (+/- 27)
test writer::bench_vec_writer_1_inline_always               ... bench: 50 |   [-***#******--]                        | 200:        75 ns/iter (+/- 46)
test writer::bench_vec_writer_1_inline_never                ... bench: 500 |   [--***#***---]                       | 2000:       775 ns/iter (+/- 433)
```

Unfortunately it's getting pretty late, so rather than wait until the next time
to dive into this, I'll leave it up to you all. Does anyone know why `reserve`
is causing so much trouble here?

---

PS: While I was working on this, I saw
[stevencheg](https://github.com/erickt/rust-serialization-benchmarks/pull/2)
submitted a patch to speed up the protocol buffer support. But when I ran the
tests, everything was about 40% slower than the last benchmark
[post](http://erickt.github.io/blog/2014/11/13/benchmarks-2/)! Something
happened with Rust's performance over these past couple weeks!

```
test goser::bincode::bench_decoder                          ... bench:      7682 ns/iter (+/- 3680) = 52 MB/s
test goser::bincode::bench_encoder                          ... bench:       516 ns/iter (+/- 265) = 775 MB/s
test goser::bincode::bench_populate                         ... bench:      1504 ns/iter (+/- 324)
test goser::capnp::bench_deserialize                        ... bench:       251 ns/iter (+/- 140) = 1784 MB/s
test goser::capnp::bench_deserialize_packed_unbuffered      ... bench:      1344 ns/iter (+/- 533) = 250 MB/s
test goser::capnp::bench_populate                           ... bench:       663 ns/iter (+/- 236)
test goser::capnp::bench_serialize                          ... bench:       144 ns/iter (+/- 37) = 3111 MB/s
test goser::capnp::bench_serialize_packed_unbuffered        ... bench:       913 ns/iter (+/- 436) = 369 MB/s
test goser::msgpack::bench_decoder                          ... bench:      3411 ns/iter (+/- 1837) = 84 MB/s
test goser::msgpack::bench_encoder                          ... bench:       961 ns/iter (+/- 477) = 298 MB/s
test goser::msgpack::bench_populate                         ... bench:      1564 ns/iter (+/- 453)
test goser::protobuf::bench_decoder                         ... bench:      3116 ns/iter (+/- 1485) = 91 MB/s
test goser::protobuf::bench_encoder                         ... bench:      1220 ns/iter (+/- 482) = 234 MB/s
test goser::protobuf::bench_populate                        ... bench:       942 ns/iter (+/- 836)
test goser::serialize_json::bench_decoder                   ... bench:     31934 ns/iter (+/- 16186) = 18 MB/s
test goser::serialize_json::bench_encoder                   ... bench:      8481 ns/iter (+/- 3392) = 71 MB/s
test goser::serialize_json::bench_populate                  ... bench:      1471 ns/iter (+/- 426)
```
