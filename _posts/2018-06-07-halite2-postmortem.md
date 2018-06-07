---
layout: post
title: Halite II Postmortem
date: 2018-06-07
description: A brief overview of my Halite II experience.
category: [contest]
---

I first heard about [Halite II][1], an AI competition hosted by TwoSigma,
from my apartment-mate [@jhuang97][2]. The premise is simple:
harvest resources from planets to manufacture more ships,
destroy the enemy, and be the last fleet standing. I managed to place a
respectable 57th in the final standings, and 15th among university students.
[Here's my bot's source code.][0]

##### **Motivation**

Originally, I wasn't intending to become this invested in Halite. I had just been
reading about the new-ish systems programming language [Rust][3], and figured this
could be a good opportunity to try it out. But once I had a minimally working bot
(read as: doesn't crash immediately), I couldn't resist the satisfaction of
climbing the leaderboard.

##### **Code Architecture**

My codebase is split into the following categories:

- [State][4]: representing the current game state
- [Parsing][5]: communicating with the Halite client
- [Navigation][6]: collision detection and resolution
- [Scouting][7]: analyzing the current game state
- [Strategy][8]: high-level decision making

I opted early on for a stateless algorithm, where each turn's decisions are
independent of the previous turn. This works a little better with Rust's functional
flavor, and also makes things easier to reason about.

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
usage of `planet.owner` is checked at compile time. We can't just forget to check
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
new instance of the data type. Once we've defined our own trait, we can implement it
for any type: even types from the standard library, like `i32` and `f32`.

###### **Navigation**

Collision detection was the most math-heavy section of my codebase.

For broad-phase detection, I used a sparse grid. Each entity is hashed into
one or more rectangular bins. When checking for collisions, only entities in
the same bin need to be checked against each other. I couldn't think of a good
way to check what rectangles a circle overlaps, so I computed it naively. Given
a circle:

1. Retrieve the circle's extreme coordinates (e.g. top, bottom, left, right)
2. Iterate through the entire square in bin-sized chunks
3. Check if circle overlaps bin

This works well for small circles, where we don't have to iterate through many
bins. Since this is typically the case when checking collisions between ships
(which are small relative to bin size), it was good enough for me.

For narrow-phase detection, I used vector operations to check intersections
between circles and lines.

As far as actual navigation goes, a naive approach served me pretty well: I sent my
ships in straight lines toward their targets, and adjusted course until there were
no collisions.

###### **Scouting**

Although everyone has complete knowledge of the game state, there's still a lot of information
to extract. Most of my calculations were concerned with distance: for example, where are the
nearest allies, enemies, and planets? Do I have more allies in my combat radius than enemies?
Most functions consist of a series of filters on the global population, and look something like this:

```rust
pub fn nearest_target(&self, ship: &Ship, d: f64) -> Option<&Ship> {
    self.ships[&ship.id].iter()
        .take_while(|other| ship.distance_to(other) < d)
        .filter(|other| other.owner != ship.owner)
        .filter(|other| other.is_docked())
        .next()
}
```

Method chaining and closures give a distinctly functional feel to this part of the codebase.

###### **Strategy**

This is where I actually decide on commands for each ship. I thought of two ways to structure
this logic:

1. As an omniscient general, dictating orders based on global need
2. As a series of single ships, choosing actions based on local need

I found the second approach easier to think through, so that's what I went with. Each ship
has the following priorities:

- Close range
  1. Retreat when outnumbered
  2. Attack when enemies are nearby
  3. Reinforce when nearby allies are retreating

- Long range
  1. Dock at the closest planet
  2. Reinforce allies
  3. Attack the closest enemy

Close-range reactions were prioritized over long-range goals.

##### **GIF**

This is the last iteration of my bot playing against itself:

![gif](/assets/halite.gif)

You can see how clustering naturally occurs as ships rally to their allies when outnumbered.

##### **Notes**

There are a lot of improvements to docking logic that I didn't have time to implement. Docking
is highly risky, since it leaves your ship completely vulnerable for at least the 10 turns
required to dock and undock. I never found a good metric for "docking safety," although I tried:

- Number of enemies in a given radius around the ship: too conservative
- Ratio of enemies to allies around the ship: too conservative with high number of enemies
- Difference between allies and enemies: too eager with high number of enemies

Additionally, parameters like

- How far to search for a docking spot
- How far to search for retreating allies
- When to chase an enemy

Were all decided mostly by trial and error. I suppose a machine-learning algorithm could do better
at fitting parameters, but I went with my intuition--maybe next time?

I tested bots almost entirely locally, pitting newer versions against older counterparts. This
turned out to be not great, since I wasn't introducing my bots to new opponent strategies. Sometimes
a bot would win 99% of the time against an old version, but do much worse on the real leaderboard,
typically because it just happened to exploit a weakness in the old bot that real opponents didn't have.

##### **Discarded Ideas**

Too many to list, but here are a few:

- Entirely parameter-based approach, where action is scored by some objective function

Discarded because I couldn't find a satisfying objective function

- Additional tactics module, to micro ships that had been given more general orders

Discarded because the complexity didn't really improve the winrate

- Distraction task, where a single ship would try to lure multiple enemy ships away

Discarded because most enemies didn't chase very far

##### **Credits and Inspiration**

The Halite II Discord server was a great opportunity to chat with other players, and quite a few
of the top players were active in the chat. In particular, I'd like to point out the following people:

- [jhuang97][2], who introduced me to Halite
- [FrankWhoee][9], a friendly high schooler that blew past me to 33rd place
- [Spelljack][10], a PhD friend with ridiculously simple Python code that landed him 12th place
- [Fohristiwhirl][11], who developed an [open-source replay viewer][12] and an incredibly sophisticated [early game analysis][13]

[0]: https://github.com/nwtnni/halite
[1]: https://halite.io/
[2]: https://github.com/jhuang97/Halite2
[3]: https://www.rust-lang.org/en-US/
[4]: https://github.com/nwtnni/halite/blob/master/src/hlt/state.rs
[5]: https://github.com/nwtnni/halite/blob/master/src/hlt/parse.rs
[6]: https://github.com/nwtnni/halite/blob/master/src/hlt/collision.rs
[7]: https://github.com/nwtnni/halite/blob/master/src/hlt/scout.rs
[8]: https://github.com/nwtnni/halite/blob/master/src/hlt/strategy.rs
[9]: https://github.com/FrankWhoee/Halite2Bot
[10]: https://github.com/spelljack/haliteII
[11]: https://github.com/fohristiwhirl/gohalite2 
[12]: https://github.com/fohristiwhirl/chlorine
[13]: https://github.com/fohristiwhirl/halite2_rush_theory
