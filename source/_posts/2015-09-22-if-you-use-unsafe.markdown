---
layout: post
title: "If you use unsafe, you should be using compiletest"
date: 2015-09-22 10:58:25 -0700
comments: true
categories: [rust, unsafe, compiletest]
---

One of the coolest things about the Rust typesystem is that you can use it to
make unsafe bindings safe. Read all about it in the
[Rustonomicon](https://doc.rust-lang.org/nightly/nomicon/). However, it can be
really quite easy to slip in a bug where you're not actually making the
guarantees you think you're making. For example, here's a real bug I made in
the [ZeroMQ FFI bindings](https://github.com/erickt/rust-zmq) (which have been
edited for clarity):

```rust
pub struct Socket {
    sock: *mut libc::c_void,
    closed: bool
}

impl Socket {
    pub fn as_poll_item<'a>(&self, events: i16) -> PollItem<'a> { // <- BUG!!!
        PollItem {
            socket: self.sock,
            fd: 0,
            events: events,
            revents: 0,
            marker: PhantomData
        }
    }
}

impl Drop for Socket {
    fn drop(&mut self) {
        unsafe {
            zmq_sys::zmq_close(self.sock);
        }
    }
}

pub struct PollItem<'a> {
    socket: *mut libc::c_void,
    fd: libc::c_int,
    events: i16,
    revents: i16,
    marker: PhantomData<&'a Socket>
}

pub fn poll(items: &mut [PollItem], timeout: i64) -> Result<i32, Error> {
    unsafe {
        let rc = zmq_sys::zmq_poll(
            items.as_mut_ptr() as *mut zmq_sys::zmq_pollitem_t,
            items.len() as c_int,
            timeout as c_long);

        if rc == -1i32 {
            Err(errno_to_error())
        } else {
            Ok(rc as i32)
        }
    }
}
```

Here's the bug if you missed my callout:

```rust
    pub fn as_poll_item<'a>(&self, events: i16) -> PollItem<'a> { // <- BUG!!!
```

My intention was to tie the lifetime of `PollItem<'a>` to the lifetime of the
`Socket`, but because I left out one measly `'a`, Rust doesn't tie the two
together, and instead is actually using the `'static` lifetime. This then lets
you do something evil like:


```
// leak the pointer!
let poll_item = {
		let context = zmq::Context::new();
		let socket = context.socket(zmq::PAIR).unwrap();
		socket.as_poll_item(0)
};

// And use the now uninitialized pointer! Wee! Party like it's C/C++!
poll(&[poll_item], 0).unwrap();
```

It's just that easy. Fix is simple, just change the function to use `&'a self`
and Rust will refuse to compile this snippet. Job well done!

Well, no, not really. Because what was particularly devious about this bug is
that it actually came back. Later on I accidentally reverted `&'a self` back to
`&self` because I secretly hate myself. The project and examples still compiled
and ran, but that unitialized dereference was just waiting around to cause a
security vulnerability.

Oops.

Crap.

Making sure Rust actually rejects programs that it ought to be rejecting
**fundamentally important** when writing a library that uses Unsafe Rust.

That's where [compiletest](https://github.com/laumann/compiletest-rs) comes in.
It's a testing framework that's been extacted from 
[rust-lang/rust](https://github.com/rust-lang/rust)
that lets you write these "shouldn't-compile" tests. Here's how to use it.
First add this to your `Cargo.toml`. We do a little feature dance because
currently `compiletest` only runs on nightly:

```toml
...
[features]
unstable = ["compiletest_rs"]
...

[dependencies]
compiletest_rs = { "version = "*", optional = true }
...
```

Then, add add a test driver `tests/compile-tests.rs` (or whatever you want to
name it) that runs the compiletest tests:

```rust
#![cfg(feature = "unstable")]

extern crate compiletest_rs as compiletest;

use std::path::PathBuf;
use std::env::var;

fn run_mode(mode: &'static str) {
    let mut config = compiletest::default_config();

    let cfg_mode = mode.parse().ok().expect("Invalid mode");

    config.target_rustcflags = Some("-L target/debug/ -L target/debug/deps/".to_owned());
    if let Ok(name) = var::<&str>("TESTNAME") {
        let s : String = name.to_owned();
        config.filter = Some(s)
    }
    config.mode = cfg_mode;
    config.src_base = PathBuf::from(format!("tests/{}", mode));

    compiletest::run_tests(&config);
}

#[test]
fn compile_test() {
    run_mode("compile-fail");
}
```

Finally, add the test! Here's the one I wrote, `tests/compile-fail/no-leaking-poll-items.rs`:

```rust
extern crate zmq;

fn main() {
    let mut context = zmq::Context::new();
    let _poll_item = {
        let socket = context.socket(zmq::PAIR).unwrap();
        socket.as_poll_item(0) //~ ERROR error: `socket` does not live long enough
    };
}
```

Now you can live in peace with the confidence that this bug won't ever appear again:

```
% multirust run nightly cargo test --features unstable
     Running target/debug/compile_tests-335c5f56b353961f

running 1 test

running 1 test
test [compile-fail] compile-fail/no-leaking-poll-items.rs ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured

test compile_test ... ok
```

In summary, use `compiletest`, and demand it's use from the Unsafe Rust
libraries you use! Otherwise you can never be sure if unsafe and undefined
behavior like this will sneak into your project.

TLDR:

{% img center /images/compiletest-badtime.jpg Bad Time %}
