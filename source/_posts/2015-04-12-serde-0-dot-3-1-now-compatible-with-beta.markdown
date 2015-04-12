---
layout: post
title: "serde 0.3.1 - now compatible with beta! plus aster and quasi updates"
date: 2015-04-12 11:53:43 -0700
comments: true
categories: [rust, serialization, serde, aster, quasi]
---

I just pushed up [serde](https://github.com/erickt/rust-serde) 0.3.1 to
[crates.io](https://crates.io/crates/serde), which is now compatible with beta!
serde\_macros 0.3.1, however still requires nightly.  But this means that if
you implement the all the traits using stable features, then any users of serde
should work with rust 1.0.

Here's what's also new in serde v0.3.1:

* Renamed `ValueDeserializer::deserializer` into `ValueDeserializer::into_deserializer`.
* Renamed the attribute that changes the name a field is serialized
  `#[serde(alias="...")]` to `#[serde(rename="...")]`.
* Added implementations for `Box`, `Rc`, and `Arc`.
* Updated `VariantVisitor` to hint to the deserializer which variant kind it is expecting.
  This allows serializers to serialize a unit variant as a string.
* Added an `Error::unknown_field_error` error message.
* Progress on the documentation, but there's still plenty more to go.

Upstream of serde, I've been also doing some work on
[aster](https://github.com/erickt/rust-aster) and
[quasi](https://github.com/erickt/rust-quasi), which are my helper libraries to
simplify writing syntax extensions.

aster v0.2.0:

* Added builders for qualified paths, slices, `Vec`, `Box`, `Rc`, and `Arc`.
* Extended item builders to support `use` simple paths, globs, and lists.
* Added a helper for building the `#[automatically_derived]` annotation.

quasi v0.1.9:

* Backported support for `quote_attr!()` and `quote_matchers!()` from `libsyntax`.
* Added support for unquoting arbitrary slices.

Thanks for everyone's help with this release!
