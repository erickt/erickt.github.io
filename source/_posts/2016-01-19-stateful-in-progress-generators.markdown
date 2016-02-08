---
layout: post
title: "Stateful: a Rust experimental syntax extension for generators and more"
date: 2016-01-27 08:31:27 -0800
comments: true
categories: [rust, syntax extensions, stateful, generators, async/await]
---

AKA: Erick Does More Horrible Things to Rust

Hello internet! It's been too long. Not only are the
[Rust Meetups](http://www.meetup.com/rust-bay-area) back up and running, it's
time for me to start back to blogging. For the past couple months, I've been
working on a new syntax extension that will allow people to create fun and
exciting new control flow mechanisms in stable Rust.  "For the love of all that
is sigils, why?!"  Well, Because I can.  Sometimes when you stare into the
madness, it stares back into you? Or something like that?

It's called [Stateful](https://github.com/erickt/stateful), which helpfully has
no documentation. Such an innocent name, right?  It's very much in progress
(and mostly broken) implementation of some of the ideas in this and future
posts.  So don't go and think these code snippets are executable just yet :)

Anyway, lets show off `Stateful` by showing how we can implement
[Generators](https://en.wikipedia.org/wiki/Generator_%28computer_programming%29).
We've got an [RFC ticket](https://github.com/rust-lang/rfcs/issues/388) to
implement them, but wouldn't it be nice to have them sooner?  For those of you
unfamiliar with the concept, Generators are function that can be returned from
multiple times, all while preserving state between those calls.  Basically,
they're just a simpler way to write
[Iterators](https://doc.rust-lang.org/std/iter/#iterator).

Say we wanted to iterate over the numbers 0, 1, and 2.  Today, we would write
an `Iterator` with something like this:

```rust
struct Iter3(usize);

impl Iter3 {
    fn new() -> Self {
        Iter3(0)
    }
}

impl Iterator for Iter3 {
    fn next(&mut self) -> Option<usize> {
        if self.0 < 3 {
            let i = self.0;
            self.0 += 1;
            Some(i)
        } else {
            None
        }
    }
}

fn main() {
    let iter = Iter3::new();
    assert_eq!(iter.next(), Some(0));
    assert_eq!(iter.next(), Some(1));
    assert_eq!(iter.next(), Some(2));
    assert_eq!(iter.next(), None);
}
```

The struct preserves our state across these function calls.  It's a pretty
straightforward implementation, but it does have some amount of boilerplate
code. For large iterator implementations, this state management can get quite
complicated.  Instead, lets see how this same code could be expressed with
something like `Stateful`:

```rust
#![plugin(stateful)]

#[generator]
fn gen3() -> Iterator<Item=usize> {
    let mut i = 0;
    while i < 3 {
        yield_!(i);
        i += 1;
    }
}
```

Where `yield_!(i)` is some magical control flow mechanism that not only
returned some value `Some(i)`, but also made sure on the `iter.next()` would
jump the execution to just after the yield.  At the end of the generator, we'd
just return `None`.  We could simplify this even more by unrolling that loop
into:

```rust
#[generator]
fn gen3_unrolled() -> Iterator<Item=usize> {
    yield_!(0);
    yield_!(1);
    yield_!(2);
}
```

The fun part is figuring out how to convert these generators into something
that's roughly equivalent to `Iter3`.  At it's heart, `Iter3` really is a
simple state machine, where we save the counter state in the structure before
we "yield" the value to the caller.  Let's look at what we would generate for
`gen3_unrolled`.

First, we need some boilerplate, that sets up the state of our generator.  We
don't yet have [impl
trait](https://aturon.github.io/blog/2015/09/28/impl-trait/), so we hide all
our stuff in a module:

```rust
fn gen3_unrolled() -> gen3_unrolled::Generator {
    gen3_unrolled::Generator::new()
}

mod gen3_unrolled {
    pub struct Generator {
        state: State,
    }

    impl Generator {
        pub fn new() -> Self {
            Generator {
                state: State::Enter,
            }
        }
    }

    ...
```

We represent our generator's state with an enum.  We have our initial state, a
state per yield, then an exit state:

```rust
    enum State {
        Enter,
        AfterYield0,
        AfterYield1,
        AfterYield2,
        Exit,
    }
```

Finally, we have our state machine, and a pretty trivial `Iterator`
implementation that manages entering and exiting the state machine:

```rust
    impl Iterator for Generator {
        type Item = usize;

        fn next(&mut self) -> Option<usize> {
            let state = mem::replace(&mut self.state, State::Exit);
            let (result, next_state) = advance(state);
            self.state = next_state;
            result
        }
    }

    fn advance(mut state: State) -> (Option<usize>, State) {
        loop {
            state = match state {
                State::Enter => {
                    return_!(Some(0); State::AfterYield0);
                }
                State::AfterYield0 => {
                    return_!(Some(1); State::AfterYield1);
                }
                State::AfterYield1 => {
                    return_!(Some(2); State::AfterYield2);
                }
                State::AfterYield2 => {
                    goto!(State::Exit);
                }
                State::Exit => {
                    return_!(None; State::Exit);
                }
            }
        }
    }
}
```

We move the current `state` into `advance`, then have this `loop-match` state
machine.  Then there are 2 new control flow constructs:
`return_!($expr; $next_state)` and our old friend `goto!($next_state)`.
`return_!()` returns some value and also sets the position the generator should
resume at, and `goto!()` just sets the next state without leaving the function.

Here's one way they might be implemented:

```rust
macro_rules! goto {
    ($next_state:expr) => {
        $state = $next_state;
        continue;
    }
}

macro_rules! return_ {
    ($result: expr; $next_state:expr) => {
        return ($result, $next_state);
    }
}
```

Relatively straightforward transformation, right?  But that's an easy case.
Things start to get a wee bit more complicated when we start thinking about how
we'd transform `gen3`, because it's got both a `while` loop and a mutable
variable.  Lets see that in action.  I'll leave out the boilerplate code and
just focus on the `advance` function:

```rust
fn advance(mut state: State) -> (Option<usize>, State) {
    loop {
        match state {
            State::Enter => {
                let mut i = 0;
                goto!(State::Loop(i));
            }
            State::Loop(mut i) => {
                if i < 3 {
                    goto!(State::Then(i));
                } else {
                    goto!(State::Else(i));
                }
            }
            State::Then(mut i) => {
                return_!(Some(i); State::AfterYield(i));
            }
            State::Else(mut i) => {
                goto!(State::AfterLoop(i));
            }
            State::AfterYield(mut i) => {
                i += 1;
                goto!(State::Loop(i));
            }
            State::AfterLoop(mut i) => {
                goto!(State::Exit);
            }
            State::Exit => {
                return_!(None; State::Exit);
            }
        }
    }
}
```

Now things are getting interesting! There are two critical things we can see
off the bat.  First, we need to reify the loops and conditionals into the state
machine, because they affect the control flow.  Second, we need to lift any
variables that are accessed across states into the `State` enum.

We can also start seeing the complications.  The obvious one is mutable
variables.  We need to somehow thread the information about `i`'s mutability
through each of the states.  This naive implementation would trip over the
`#[warn(unused_mut)]` lint.  And now you might start to get a sense of the
horror that lies beneath `Stateful`.

At this point, you might be thinking to yourself, "Self, if mutable variables
are going to be complicated, what about copies and moves?"  You sound like a
pretty sensible person.  Therein lies madness.  You might want to stop thinking
too deeply on it.  If you can't, maybe you think "Wait.
What about Generics?" Yep.  "Borrows?!"  Now I'm getting a little worried.
"How do you even know what's a variable!?!"  Sorry.

Yeah so there are one or two things that might be a tad challenging.

---

So that's `Stateful`.  It's an experiment to get some real world experience
with these control flow mechanisms that may someday feed into RFCs, and maybe,
just maybe, might get implemented in the compiler.  There's no reason we need
to support everything, which would require us to basically reimplement the
compiler.  Instead, I believe there's a subset of Rust that we *can* support in
order to start getting real experience now.

Generators area really just the start.  There's a whole host of other things
that really are just other things that, if you just squint at em, are really
just state machines in disguise.  It's quite possible if we can pull
`Stateful`, we'll also be able to implement things like
[Coroutines](https://en.wikipedia.org/wiki/Coroutine),
[Continuations](https://en.wikipedia.org/wiki/Continuation-passing_style), and
that hot new mechanism all the cool languages are implementing these days,
[Async/Await](https://en.wikipedia.org/wiki/Await).

But that's all for later.  First is to get this to work.  In closing, I leave
you with these wise words.

> ph'nglui mglw'nafh Cthulhu R'lyeh wgah'nagl fhtagn.
