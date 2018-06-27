---
layout: post
title: Battlecode 2018 Postmortem
date: 2018-06-07
description: A brief overview of my Battlecode 2018 experience, using spacetime A\* and the Hungarian assignment algorithm.
category: [contest]
---

After competing in [Halite II][1], I was excited to participate in [Battlecode 2018][2].
I teamed up with [jhuang97][3] and [Spelljack][4], and we placed in the top 16 nationally
as team **Fresh Huang**, unfortunately missing the live finals at MIT by one match.
Here I describe the technically interesting parts of our bot: spacetime A\* for navigation,
and the Hungarian algorithm for objective assignment. [Here's the source code for our bot.][5]

##### **Background**

Battlecode, hosted by MIT, is one of the world's more well-known AI 
programming competitions. Think RTS game--e.g. Starcraft--but writing a 
program to control your units instead of manually interacting with them.
To write a successful AI, you have to consider:

- Limited map vision
- Unknown terrain
- Resource management
- Resource gathering
- Combat micro
- Construction priority
- Runtime constraints

...and more.

##### **Our Bot**

To be honest, our strategy was fairly naive. Our main loop looks a lot like this:

```rust
for worker in workers {
    if worker.can_build() && worker.is_safe() {
        worker.build()
    }

    if worker.can_harvest() && !factories.is_empty() {
        worker.harvest()
    }

    // And so on
}

for knight in knights {
    if knight.can_attack() && knight.has_allies() {
        knight.attack()
    }

    // And so on
}

// More units
```

In fact, most of our main loop didn't change from our very first implementation.
Not because this is excellent AI code, by any measure--but because we
didn't have the time to write better micro or cleaner code. The majority of
our codebase was held together by `&&`'s, `||`'s, magic numbers, and hopes and dreams.
So let's move on to the interesting stuff.

##### **Pathfinding**

Although the maps are grid-based, pathfinding was surprisingly difficult for a number
of reasons:

1. Movement heat: units can't move every turn.
2. Unit collision: units can't pass through each other.
3. Non-simultaneous resolution: commands are executed in sequence.
4. Non-connected maps: not all tiles are reachable.

A long time ago, I read [this RougeBasin article][6] on "Djikstra maps." The idea
is simple: run Djikstra's algorithm from the destination instead of the source,
and cache the distance to every point. This creates a flow map that any unit
can ussearche to travel to the destination in constant time: just go downhill. That
is, check all eight directions, see which one has the lowest value, and take
a step.

![GIF of normal Djikstra's](/assets/battlecode-djisktra.gif)

![GIF of worker pathing](/assets/battlecode-worker.gif)

[1]: https://halite.io/
[2]: http://battlecode.org/
[3]: https://github.com/jhuang97
[4]: https://github.com/spelljack
[5]: https://github.com/nwtnni/battlecode 
[6]: http://www.roguebasin.com/index.php?title=The_Incredible_Power_of_Dijkstra_Maps
