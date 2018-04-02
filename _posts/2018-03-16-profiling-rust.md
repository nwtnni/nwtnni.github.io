---
layout: post
title: Profiling Rust
date: 2018-03-16
description: A beginner's introduction to profiling Rust programs.
category: [project, hungarian]
---

After publishing my first crate, an implementation of the [Hungarian algorithm][1],
I found that an existing crate was much faster--even though my codebase was about
five times smaller. Armed with some other beginner-friendly blog posts on profiling,
I set out to figure out why my code was so slow.

##### **Background**

The Hungarian (or Kuhn-Munkres) is an `O(n^3)` algorithm for solving the optimal
assignment problem. For example, let's say I have three workers, and three jobs.
Exactly one worker needs to be assigned to each job, and they all charge different
prices per job. We can represent this as a table (or matrix):

|          | Job 1 | Job 2 | Job 3 |
|:--------:|------:|------:|------:|
| Worker 1 | 5     | 10    | 20    |
| Worker 2 | 1     | 12    | 50    |
| Worker 3 | 6     | 25    |  2    |

Where each entry (i, j) represents the cost for Worker i to do Job j. So this is
the optimal assignment problem: find the assignment of workers to jobs that minimizes
the total cost.

Naively, this would take `O(n!)` time: we'd have to check every possible pairing.
Worker 1 has 3 options, then worker 2 has 2 options, and then worker 3 only has 1 option,
giving a total of `3 * 2 * 1 = 3! = 6` possible pairings. The Hungarian algorithm
solves this problem much more efficiently.

##### **Motivation**

I first discovered this algorithm when competing in [Battlecode 2018][2]. Given
a set of resource deposits and workers, I wanted to find an assignment of workers
to minimize the total travel time to deposits. After many failed attempts to implement
the [Wikipedia description][3], I finally found this [excellent explanation][4],
which all of the implementations I've found seem to based off of.

But the code seemed unnecessarily verbose: did we really need all of those functions?
I set out to write a simplified version, and eventually arrived at a single equivalent
function, less than 150 lines long. [Here's the final result][5]: the comments
map sections of code to the original explanation. I collected tests and examples
from wherever I could to make sure my translation retained its correctness.

I had benchmarks for my own code (thanks to [Criterion.rs][5]), but I didn't benchmark
any reference implementations, and I couldn't find any benchmarking data
for existing versions. Since the code was so much simpler,
I just assumed it would be faster. And so I published version 0.1.0 to Crates.io.

##### **Profiling**

A day or so later, I decided to benchmark the other Rust implementation,
[munkres-rs][6]. I found two others ([][7])

[1]: https://en.wikipedia.org/wiki/Hungarian_algorithm
[2]: http://battlecode.org/
[3]: https://en.wikipedia.org/wiki/Hungarian_algorithm#Matrix_interpretation
[4]: http://csclab.murraystate.edu/~bob.pilgrim/445/munkres.html
[5]: https://github.com/japaric/criterion.rs
[6]: https://github.com/mneumann/munkres-rs
[7]: https://github.com/saebyn/linear-assignment-rust/
