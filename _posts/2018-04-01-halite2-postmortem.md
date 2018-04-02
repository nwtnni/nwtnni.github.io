---
layout: post
title: Halite II Postmortem
date: 2018-04-01
description: A brief overview of my Halite II experience, with some snippets of Rust.
category: [project, contest]
---

[Here's my bot's source code.][0]
I first heard about [Halite II][1], an AI competition hosted by TwoSigma,
from my apartment-mate [@jhuang97][2]. The premise is simple:
harvest resources from planets to manufacture more ships,
destroy the enemy, and be the last fleet standing. I managed to place a
respectable 57th in the final standings, and 15th among university students.

##### **Motivation**

Originally, I wasn't intending to become this invested in Halite. I had just been
reading about the new-ish systems programming language [Rust][3], and figured this
could be a good opportunity to try it out. But once I had a minimally working bot
(read as: doesn't crash immediately), I couldn't resist the satisfaction of
climbing the leaderboard.

##### **Code Architecture**

My codebase is split into the following categories:

- [State][4]: representing the game state
- [Parsing][5]: communicating with the Halite client
- [Navigation][6]: collision detection and resolution
- [Scouting][7]: analyzing the current game state
- [Strategy][8]: high-level decision making

I opted early on for a stateless approach, where each turn's decisions are
independent of the previous. This works a little better with Rust's functional 
flavor, and also makes the algorithm easier to reason about.

###### **state.rs**

Game entities like ships and planets were easily represented by structs, mapping
fields almost one-to-one to the original Halite descriptions. The only
notable thing here is getting to use the `Option` enum to describe
nullable values. For example, not all planets have owners:

```rust
// With special value
pub fn is_owned(planet: &Planet) -> bool {
    return planet.owner != 0  
}

// With Option
pub fn is_owned(planet: &Planet) -> bool {
    match planet.owner with {
    | Some(_) => true,
    | None    => false,
    }
}
```

While the second implementation is a little more verbose, the benefit is that our
usage of `planet.owner` is checked at *compile time*. We can't just forget to check
for the special value anymore: `Option<ID>` is a completely different type than `ID`,
and the compiler enforces that we handle the `None` case.

###### **Parsing**

The Halite server communicates via `stdin` and `stdout`. The communication protocol
was designed so parsing could be done in a streaming fashion: all variable length
sections are preceded by a count. I used a trait to encode this:

```rust
pub trait FromStream {
    fn take(stream: &mut Vec<&str>) -> Self;
}
```

We can read this as: all data types implementing the `FromStream` trait define a function
`take`, which takes in a mutable queue of strings (the stream) and returns a 
new instance of the data type. Once we've defined a trait like this,

[0]: https://github.com/nwtnni/halite
[1]: https://halite.io/
[2]: https://github.com/jhuang97/Halite2
[3]: https://www.rust-lang.org/en-US/
[4]: https://github.com/nwtnni/halite/blob/master/src/hlt/state.rs
[5]: https://github.com/nwtnni/halite/blob/master/src/hlt/parse.rs
[6]: https://github.com/nwtnni/halite/blob/master/src/hlt/collision.rs
[7]: https://github.com/nwtnni/halite/blob/master/src/hlt/scout.rs
[8]: https://github.com/nwtnni/halite/blob/master/src/hlt/strategy.rs
