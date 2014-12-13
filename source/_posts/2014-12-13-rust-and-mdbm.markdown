---
layout: post
title: "Using Rust to make a safer interface for Yahoo's Fast MDBM database"
date: 2014-12-13 11:09:19 -0800
comments: true
categories: [rust, database, mdbm]
---

I'm really supposed to be working on my
[serialization series](http://erickt.github.io/blog/categories/serialization/),
 but I just saw a neat new library that was open sourced by Yahoo a couple days
ago called [MDBM](https://github.com/yahoo/mdbm) on
[Hacker News](https://news.ycombinator.com/item?id=8732891). I know nothing
about the library, but there are some really neat claims:

1. It's supposed to be fast, and that's always nice.
2. It's supposed to have a really slim interface.
3. It's so fast because it's passing around naked pointers to mmapped files,
   which is terribly unsafe. Unless you got rust which can prove that those
   pointers won't escape :)

So I wanted to see how easy it'd be to make a Rust binding for the project.
If you want to follow along, first make sure you have
 [rust installed](http://www.rust-lang.org/install.html). Unfortunately it
looks like MDBM only supports Linux and FreeBSD, so I had to build out a Fedora
VM to test this out on. I *think* this is all you need to build it:

```
% git clone https://github.com/yahoo/mdbm
% cd mdbm/redhat
% make
% rpm -Uvh ~/rpmbuild/RPMS/x86_64/mdbm-4.11.1-1.fc21.x86_64.rpm
% rpm -Uvh ~/rpmbuild/RPMS/x86_64/mdbm-devel-4.11.1-1.fc21.x86_64.rpm
```

Unfortunately it's only for linux, and I got a mac, but it turns out there's
plenty I can do to prep while VirtualBox and Fedora 21 download. Lets start out
by creating our project with [cargo](https://crates.io):

```
% cargo new mdbm
% cd rust-mdbm
```

(Right now there's no way to have the name be different than the path, so edit
`Cargo.toml` to rename the project to `mdbm`. I filed
[#1030](https://github.com/rust-lang/cargo/issues/1030) to get that
implemented).

By convention, we put bindgen packages into `$name-sys`, so make that crate as
well:

```
% cargo new --no-git mdbm-sys
% cd mdbm-sys
```

We've got a really cool tool called
[bindgen](https://github.com/crabtw/rust-bindgen), which uses clang to parse
header files and convert them into an unsafe rust interface. So lets check out
MDBM, and generate a crate to wrap it up in.

```
% cd ../..
% git clone git@github.com:crabtw/rust-bindgen.git
% cd rust-bindgen
% cargo build
% cd ..
% git clone git@github.com:yahoo/mdbm.git
% cd rust-mdbm/mdbm-sys
% DYLD_LIBRARY_PATH=/Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/lib \
  ~/rust/rust-bindgen/target/bindgen \
  -lmdbm \
  -o src/lib.rs \
  mdbm/include/mdbm.h
```

Pretty magical. Make sure it builds:

```
% cargo build
/Users/erickt/rust-mdbm/mdbm-sys/src/lib.rs:3:21: 3:25 error: failed to resolve. Maybe a missing `extern crate libc`?
/Users/erickt/rust-mdbm/mdbm-sys/src/lib.rs:3 pub type __int8_t = ::libc::c_char;
                                                                  ^~~~
/Users/erickt/rust-mdbm/mdbm-sys/src/lib.rs:3:21: 3:35 error: use of undeclared type name `libc::c_char`
/Users/erickt/rust-mdbm/mdbm-sys/src/lib.rs:3 pub type __int8_t = ::libc::c_char;
                                                                  ^~~~~~~~~~~~~~
/Users/erickt/rust-mdbm/mdbm-sys/src/lib.rs:4:22: 4:26 error: failed to resolve. Maybe a missing `extern crate libc`?
/Users/erickt/rust-mdbm/mdbm-sys/src/lib.rs:4 pub type __uint8_t = ::libc::c_uchar;
                                                                   ^~~~
...
```

Nope! The problem is that we don't have the `libc` crate imported. We don't
have a convention yet for this, but I like to do is:

```
mv src/lib.rs src/ffi.rs
```

And create a `src/lib.rs` that contains:

```rust
#![allow(non_camel_case_types)]
#![allow(non_snake_case)]

extern crate libc;

type __builtin_va_list = libc::c_void;

include!("ffi.rs")
```

This lets me run bindgen later on without mucking up the library. This now
compiles. Next up is our high level interface. Add `mdbm-sys` to our high level
interface by adding this to the `rust-mdbm/Cargo.toml` file:

```toml
[dependencies.mdbm-sys]
path = "mdbm-sys"
```

By now I got my VirtualBox setup working, so now to the actual code! Lets start
with a barebones wrapper around the database:

```rust
pub struct MDBM {
    db: *mut mdbm_sys::MDBM,
}
```

Next is the constructor and destructor. I'm hardcoding things for now and using
IoError, since MDBM appears to log everything to the ERRNO:

```
impl MDBM {
    pub fn new(
        path: &Path,
        flags: uint,
        mode: uint,
        psize: uint,
        presize: uint
    ) -> Result<MDBM, IoError> {
        unsafe {
            let path = path.to_c_str();
            let db = mdbm_sys::mdbm_open(
                path.as_ptr(),
                flags as libc::c_int,
                mode as libc::c_int,
                psize as libc::c_int,
                presize as libc::c_int);

            if db.is_null() {
                Err(IoError::last_error())
            } else {
                Ok(MDBM { db: db })
            }
        }
    }

    ...
}

impl Drop for MDBM {
    fn drop(&mut self) {
        unsafe {
            mdbm_sys::mdbm_sync(self.db);
            mdbm_sys::mdbm_close(self.db);
        }
    }
}
```

Pretty straightforward translation of the examples with some hardcoded values
to start out. Next up is a wrapper around MDBM's `datum` type, which is the
type used for both keys and values. `datum` is just a simple struct containing
a pointer and length, pretty much analogous to our `&[u8]` slices. However our
slices are much more powerful because our type system can guarantee that in
safe Rust, these slices can never outlive where they are derived from:

```rust
pub struct Datum<'a> {
    bytes: &'a [u8],
}

impl<'a> Datum<'a> {
    pub fn new(bytes: &[u8]) -> Datum {
        Datum { bytes: bytes }
    }
}

fn to_raw_datum(datum: &Datum) -> mdbm_sys::datum {
    mdbm_sys::datum {
        dptr: datum.bytes.as_ptr() as *mut _,
        dsize: datum.bytes.len() as libc::c_int,
    }
}
```

And for convenience, lets add a `AsDatum` conversion method:

```rust
pub trait AsDatum for Sized? {
    fn as_datum<'a>(&'a self) -> Datum<'a>;
}

impl<'a, Sized? T: AsDatum> AsDatum for &'a T {
    fn as_datum<'a>(&'a self) -> Datum<'a> { (**self).as_datum() }
}

impl AsDatum for [u8] {
    fn as_datum<'a>(&'a self) -> Datum<'a> {
        Datum::new(self)
    }
}

impl AsDatum for str {
    fn as_datum<'a>(&'a self) -> Datum<'a> {
        self.as_bytes().as_datum()
    }
}
```

And finally, we got setting and getting a key-value. Setting is pretty
straightforward. The only fun thing is using the `AsDatum` constraints so we
can do `db.set(&"foo", &"bar", 0)` instead of
`db.set(Datum::new(&"foo".as_slice()), Datum::new("bar".as_slice()), 0)`.
we're copying into the database, we don't have to worry about lifetimes yet:

```rust
impl MDBM {
    ...

    /// Set a key.
    pub fn set<K, V>(&self, key: &K, value: &V, flags: int) -> Result<(), IoError> where
        K: AsDatum,
        V: AsDatum,
    {
        unsafe {
            let rc = mdbm_sys::mdbm_store(
                self.db,
                to_raw_datum(&key.as_datum()),
                to_raw_datum(&value.as_datum()),
                flags as libc::c_int);

            if rc == -1 {
                Err(IoError::last_error())
            } else {
                Ok(())
            }
        }
    }

    ...
```

MDBM requires the database to be locked in order to get the keys. This os
where things get fun in order to prevent those interior pointers from escaping.
We'll create another wrapper type that manages the lock, and uses RAII to
unlock when we're done. We tie the lifetime of the `Lock` to the lifetime of
the database and key, which prevents it from outliving either object:

```rust
impl MDBM {
    ...

    /// Lock a key.
    pub fn lock<'a, K>(&'a self, key: &'a K, flags: int) -> Result<Lock<'a>, IoError> where
        K: AsDatum,
    {
        let rc = unsafe {
            mdbm_sys::mdbm_lock_smart(
                self.db,
                &to_raw_datum(&key.as_datum()),
                flags as libc::c_int)
        };

        if rc == 1 {
            Ok(Lock { db: self, key: key.as_datum() })
        } else {
            Err(IoError::last_error())
        }
    }

    ...
}

pub struct Lock<'a> {
    db: &'a MDBM,
    key: Datum<'a>,
}

#[unsafe_destructor]
impl<'a> Drop for Lock<'a> {
    fn drop(&mut self) {
        unsafe {
            let rc = mdbm_sys::mdbm_unlock_smart(
                self.db.db,
                &to_raw_datum(&self.key),
                0);

            assert_eq!(rc, 1);
        }
    }
}
```

(Note that I've heard `#[unsafe_destrutor]` as used here may become unnecessary
in 1.0).

Finally, let's get our value! Assuming the value exists, we tie the lifetime of
the `Lock` to the lifetime of the returned `&[u8]`:

```rust
impl<'a> Lock<'a> {
    /// Fetch a key.
    pub fn get<'a>(&'a self) -> Option<&'a [u8]> {
        unsafe {
            let value = mdbm_sys::mdbm_fetch(
                self.db.db,
                to_raw_datum(&self.key));

            if value.dptr.is_null() {
                None
            } else {
                // we want to constrain the ptr to our lifetime.
                let ptr: &*const u8 = mem::transmute(&value.dptr);
                Some(slice::from_raw_buf(ptr, value.dsize as uint))
            }
        }
    }
}
```

Now to verify it works:

```rust
#[test]
fn test() {
    let db = MDBM::new(
        &Path::new("test.db"),
        super::MDBM_O_RDWR | super::MDBM_O_CREAT,
        0o644,
        0,
        0
    ).unwrap();

    db.set(&"hello", &"world", 0).unwrap();

    {
        // key needs to be an lvalue so the lock can hold a reference to
        // it.
        let key = "hello";

        // Lock the key. RIAA will unlock it when we exit this scope.
        let value = db.lock(&key, 0).unwrap();

        // Convert the value into a string. The lock is still live at this
        // point.
        let value = str::from_utf8(value.get().unwrap()).unwrap();
        assert_eq!(value, "world");
        println!("hello: {}", value);
    }
}
```

Which when run with `cargo test`, produces:

```
   Compiling rust-mdbm v0.0.1 (file:///home/erickt/Projects/rust-mdbm)
     Running target/rust-mdbm-98b81ab156dc1e5f

running 1 test
test tests::test_set_get ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured

   Doc-tests rust-mdbm

running 0 tests

test result: ok. 0 passed; 0 failed; 0 ignored; 0 measured
```

Next we want to make sure invalid behavior is a compile-time error. First, make
sure we don't leak the keys:

```rust
#[test]
fn test_keys_cannot_escape() {
    let db = MDBM::new(
        &Path::new("test.db"),
        super::MDBM_O_RDWR | super::MDBM_O_CREAT,
        0o644,
        0,
        0
    ).unwrap();

    db.set(&"hello", &"world", 0).unwrap();

    let _ = {
        let key = vec![1];
        db.lock(&key.as_slice(), 0).unwrap()
    };
}
```

Which errors with:

```
src/lib.rs:217:22: 217:25 error: `key` does not live long enough
src/lib.rs:217             db.lock(&key.as_slice(), 0).unwrap()
                                    ^~~
src/lib.rs:215:17: 218:10 note: reference must be valid for the expression at 215:16...
src/lib.rs:215         let _ = {
src/lib.rs:216             let key = vec![1];
src/lib.rs:217             db.lock(&key.as_slice(), 0).unwrap()
src/lib.rs:218         };
src/lib.rs:215:17: 218:10 note: ...but borrowed value is only valid for the block at 215:16
src/lib.rs:215         let _ = {
src/lib.rs:216             let key = vec![1];
src/lib.rs:217             db.lock(&key.as_slice(), 0).unwrap()
src/lib.rs:218         };
error: aborting due to previous error
```

And confirm the value doesn't leak either:

```rust
#[test]
fn test_values_cannot_escape() {
    let db = MDBM::new(
        &Path::new("test.db"),
        super::MDBM_O_RDWR | super::MDBM_O_CREAT,
        0o644,
        0,
        0
    ).unwrap();

    let _ = {
        db.set(&"hello", &"world", 0).unwrap();

        let key = "hello";
        let value = db.lock(&key, 0).unwrap();
        str::from_utf8(value.get().unwrap()).unwrap()
    };
}
```

Which errors with:

```
src/lib.rs:237:34: 237:37 error: `key` does not live long enough
src/lib.rs:237             let value = db.lock(&key, 0).unwrap();
                                                ^~~
src/lib.rs:233:17: 239:10 note: reference must be valid for the expression at 233:16...
src/lib.rs:233         let _ = {
src/lib.rs:234             db.set(&"hello", &"world", 0).unwrap();
src/lib.rs:235
src/lib.rs:236             let key = "hello";
src/lib.rs:237             let value = db.lock(&key, 0).unwrap();
src/lib.rs:238             str::from_utf8(value.get().unwrap()).unwrap()
               ...
src/lib.rs:233:17: 239:10 note: ...but borrowed value is only valid for the block at 233:16
src/lib.rs:233         let _ = {
src/lib.rs:234             db.set(&"hello", &"world", 0).unwrap();
src/lib.rs:235
src/lib.rs:236             let key = "hello";
src/lib.rs:237             let value = db.lock(&key, 0).unwrap();
src/lib.rs:238             str::from_utf8(value.get().unwrap()).unwrap()
               ...
src/lib.rs:238:28: 238:33 error: `value` does not live long enough
src/lib.rs:238             str::from_utf8(value.get().unwrap()).unwrap()
                                          ^~~~~
src/lib.rs:233:17: 239:10 note: reference must be valid for the expression at 233:16...
src/lib.rs:233         let _ = {
src/lib.rs:234             db.set(&"hello", &"world", 0).unwrap();
src/lib.rs:235
src/lib.rs:236             let key = "hello";
src/lib.rs:237             let value = db.lock(&key, 0).unwrap();
src/lib.rs:238             str::from_utf8(value.get().unwrap()).unwrap()
               ...
src/lib.rs:233:17: 239:10 note: ...but borrowed value is only valid for the block at 233:16
src/lib.rs:233         let _ = {
src/lib.rs:234             db.set(&"hello", &"world", 0).unwrap();
src/lib.rs:235
src/lib.rs:236             let key = "hello";
src/lib.rs:237             let value = db.lock(&key, 0).unwrap();
src/lib.rs:238             str::from_utf8(value.get().unwrap()).unwrap()
                                                     ...
```

Success! Not too bad for 2 hours of work. Baring bugs, this `mdbm`
library should perform at roughly the same speed as the C library, but
eliminate many very painful bug opportunities that require tools like Valgrind
to debug.
