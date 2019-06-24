+++
title = "Types in Paxos"
date = 2019-01-01
description = "Leveraging Rust's type system for correctness guarantees in Paxos."
+++

Our distributed systems course, [CS 5414][1], allows us to implement the projects
in a programming language of our choice. I've been pretty enamored with [Rust][2] this
past year, so I convinced [jmy48][3] and [ArunPidugu][4] to build our solutions
using [tokio][5]. After a semester of learning to write asynchronous, concurrent
protocols in Rust, including [three phase commit][6] and [COPS][7], I rewrote our
[Paxos implementation][8] from scratch over winter break, applying lessons 
learned from our original design mistakes.

## Background

Paxos is a distributed consensus protocol that guarantees safety in asynchronous
settings, and liveness with less than `n / 2` crash failures (where `n` is the
number of replicas).

We can use Paxos to implement fault-tolerant replicated state machines (sometimes
known as Multi-Paxos), by treating a state machine as a sequence of commands.
As long as each replica achieves consensus at every command in the sequence,
all replicas will arrive at the same final state.

## Architecture

The module structure is inspired by [raft-rs][9], which defines the following
divisions in the model:

- Consensus: the core consensus protocol

The core protocol is based entirely on ["Paxos Made Moderately Complex"][10].
Each server is divided into several message-passing actor threads: we have
`thread::acceptor`, `thread::commander`, `thread::leader`, `thread::replica`,
and `thread::scout` as in the original paper.

- Log: persistent storage for failure recovery

Stable storage is implemented in the `storage` module. We currently
implement stable storage by serializing and deserializing Rust data
to file with `serde` and `bincode`.

- State Machine: the user-defined state machine to replicate

The entire Paxos implementation is generic over the `State` trait defined
in the `state` module.

- Transport: communication over the network

There are two kinds of channels: internal channels between actor threads,
implemented in the `internal` module, and external channels between servers
implemented in the `external` module. The former are backed by `futures::sync::mpsc`
channels, and the latter by `tokio::net::TcpStream`.

## Types

One of my favorite features of Rust is its expressive type system, and I wanted
to validate as much of the message passing logic as possible using just the types.

Each actor thread defines its own set of acceptable messages using an enum. For
example:

```rust
// thread/acceptor.rs
pub enum In<C: state::Command> {
    P1A(message::P1A),
    P2A(message::CommanderID, message::P2A<C>),
}
```

```rust
// thread/leader.rs
pub enum In<C: state::Command> {
    Propose(message::Proposal<C>),
    Preempt(message::Ballot),
    Adopt(Vec<message::PValue<C>>),
    Decide(usize),
}
```

Exhaustive pattern matching checks along with strongly-typed communication
channels in `internal.rs` and `external.rs` help us to verify at compile time
that a) only relevant messages are sent to each actor, and b) all valid 
messages are handled by each actor.

Another major extension to the original project was to make the consensus
protocol generic over any state machine. Here I ran into some issues due to
trait bounds and associated types.

In general, it seems that [trait bounds on structs][11] are discouraged,
because they 'infect' any structs that contain them. However, because
I needed to access the associated `State::Command` and `State::Response`
types, pretty much every type ended up with `S: state::State` bounds.
On the other hand, the vast majority of these types are internal, so the
end-user doesn't have to worry about the trait bounds.

Also, because the [derive macro is overly conservative with trait bounds][12],
I ended up using the [derivative][13] crate instead.

I do miss [OCaml functors][14]: the ability to parameterize entire modules
on generic types would have come in handy here, because all types in
the `thread` module really should be parameterized on the same `state::State`
type.

## Testing

A chatroom program seems to be the Hello World of networking, so I wrote an
example server and client capable of writing messages and reading the message log.
I wrote a test harness using a JSON-based DSL to programatically send messages
to, start, and crash local servers. A typical test case looks something like
this:

- Start some servers
- Concurrently write to some of them
- Crash and restart servers while writing
- Read from each server to make sure they're consistent

For example, an execution could look like this:

```
Executing command Start { id: 0, port: 10000, count: 3 }
Executing command Start { id: 1, port: 10001, count: 3 }
Executing command Start { id: 2, port: 10002, count: 3 }
Executing command Sleep { ms: 1000 }
Executing command Connect { id: 0 }
Executing command Connect { id: 1 }
Executing command Connect { id: 2 }
Executing command Sleep { ms: 1000 }
Executing command Put { id: 0, message: "a" }
Executing command Put { id: 1, message: "b" }
Executing command Crash { id: 0 }
Executing command Put { id: 2, message: "c" }
Executing command Start { id: 0, port: 10000, count: 3 }
Executing command Sleep { ms: 1000 }
Executing command Connect { id: 0 }
Executing command Sleep { ms: 3000 }
Executing command Get { id: 0 }
Client 0 received message log ["b", "c"]
Executing command Get { id: 1 }
Client 1 received message log ["b", "c"]
Executing command Get { id: 2 }
Client 2 received message log ["b", "c"]
```

## Conclusions

Although I had to write a lot more code (mostly type annotations), I'm
pretty confident in the maintainability and extensibility of the library.
For example, at one point I refactored the `message::P1A` to include
an extra field, so I could implement a state reduction optimization
described in the paper. The compiler pointed out every use-site that
needed to be patched as a result of the change, and it was a relatively
painless, mechanical fix.

I was also using `async/await` to try it out on nightly Rust, but couldn't
figure out how to do some of the plumbing between old and new style Futures.
Currently the actor threads all manually implement `tokio::prelude::Future`,
but I'm planning to go back and switch to proper `async` methods at some point. 

This is the first time I've posted one of my projects somewhere visible--first
on [Reddit][15], and then on [Twitter][16]. I haven't gotten much technical
feedback (not sure if that's good or bad), but it's been a good start to
getting more involved in the community. Reminds me of when I first
began to meet people online in games like MapleStory or RuneScape.

[1]: http://www.cs.cornell.edu/courses/cs5414/2018fa/
[2]: https://www.rust-lang.org/
[3]: https://github.com/jmy48
[4]: https://github.com/ArunPidugu
[5]: https://tokio.rs/
[6]: https://en.wikipedia.org/wiki/Three-phase_commit_protocol
[7]: https://www.cs.cmu.edu/~dga/papers/cops-sosp2011.pdf
[8]: https://github.com/nwtnni/paxos
[9]: https://github.com/pingcap/raft-rs
[10]: http://paxos.systems/index.html
[11]: https://github.com/rust-lang/rust-clippy/issues/1689 
[12]: https://github.com/rust-lang/rust/issues/26925
[13]: https://github.com/mcarton/rust-derivative 
[14]: https://v1.realworldocaml.org/v1/en/html/functors.html
[15]: https://www.reddit.com/r/rust/comments/aafiub/paxos_for_replicated_state_machines/ 
[16]: https://twitter.com/nwtnni/status/1079177485319843845
