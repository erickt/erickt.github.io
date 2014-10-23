---
layout: post
title: "Rust for Ragel"
date: 2012-07-29 10:20
comments: true
categories: [rust, ragel]
---

I've written a bunch of simple parsers for Rust, and it's starting to get a
little obnoxious. So I added a Rust backend to the
[Ragel State Machine Compiler](http://www.complang.org/ragel/).
You can find my fork [here](https://github.com/erickt/ragel). I'm waiting for
Rust to stablize before I try to push it upstream.

Ragel is a rather neat way of writing simple parsers. In some ways it's pretty
similar to Lex, but Ragel also allows you execute arbitrary code at any point
in the state machine. Furthermore, this arbitrary code can manipulate the state
machine itself, so it can be used in many places you'd traditionally need a
full parser, such as properly handling parentheses.

Here's an example of a `atoi` function:

```
%%{
machine atoi;

action see_neg   { neg = true; }
action add_digit { res = res * 10 + (fc as int - '0' as int); }

main :=
    ( '-' @see_neg | '+' )? ( digit @add_digit )+
    '\n'?
;

write data;
}%%

fn atoi(data: ~str) -> option<int> {
    let mut neg = false;
    let mut res = 0;

    // Ragel assumes that it will be iterating over a value called data, but we
    // need to tell ragel where to start (p) and end (pe) parsing.
    let mut p = 0;
    let mut pe = data.len();

    // This is the current state in the state machine.
    let mut cs: int;

    write init;
    write exec;

    if neg { res = -1 * res; }

    // If we stopped before we hit one of the exit states, then there must have
    // been an error of some sort.
    if cs < atoi_first_final {
        none
    } else {
        some(res)
    }
}
```

While this is probably a bit more verbose than writing `atoi` by hand, it does
make the grammar pretty explicit, which can help keep it accurate.

Unfortunately there are some pretty severe performance issues at the moment.
Ragel supports two state machine styles, table-driven and goto-driven. My
backend uses tables, but since Rust doesn't yet support global constant
vectors, I need to malloc the state machine table on every function call. This
results in the [ragel-based url
parser](https://github.com/erickt/ragel/blob/rust/examples/rust/url.rl) being about
10 times slower than the equivalent table-based parser in OCaml. You can see
the generated code [here](https://gist.github.com/3200980).

The `goto` route could be promising to explore. We could simulate it using
mutually recursive function calls. OCaml does this. But again, since Rust
doesn't support tailcalls (and may
[ever](https://github.com/mozilla/rust/issues/217)), we could run into a stack
explosion. It may work well for small grammars though, and maybe LLVM could
optimize calls into tailcalls.

Unless I'm doing something glaringly wrong, it seems likely that we are going
to need some compiler help before these performance issues get solved.
