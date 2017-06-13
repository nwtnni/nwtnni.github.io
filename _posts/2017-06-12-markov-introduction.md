---
layout: post
title: Markov Introduction
date: 2017-06-12
category: markov
---

My second personal project on Github was a Markov chain text generator.
Right now it's fairly simple, using nothing more than a command-line
interface and parsing only a single file at a time. Also there are no
unit tests (oops). I'd like to revisit this project and add a few
more features--and unit tests.

##### **Background**

[Markov chains](https://en.wikipedia.org/wiki/Markov_chain) 
model "forgetful" processes: the probability of the next event depends only on 
the previous few events. Here's an example with English text. Suppose we're 
reading something, and these are the first few words:

```
My --> second --> personal --> project --> ?
```

We want to predict the next word. One of the simplest ideas is to
just look at the previous word: `project`. So we leaf through a 
book, and whenever we see the word `project`, we make
a note of what word comes next, and end up with a 
professional-looking histogram like this:

```
5 |----|
  |    |   
  |    | |----|
  |    | |    | |----|
  |    | |    | |    |
0 -----------------------
   is    was    failed
```

Okay cool--so what can we do with this graph? Well, we can extract
an estimate of [conditional probability](https://en.wikipedia.org/wiki/Conditional_probability). 
Out of the ten times we counted 'project', 'is' came next five times. 
Extending this logic to 'was' and 'failed', we can come up with this 
table:

| Word     | Probability |
|----------|:-----------:|
| 'is'     |    50%      |
| 'was'    |    30%      |
| 'failed' |    20%      |

So how do we predict what word comes next? Just take the highest
probability choice: 'is'. 

##### **Text Generation**

It turns out that the problem of text generation (i.e. how do we
generate text that sounds like a human wrote it?) can be solved by
an almost identical approach. Let's revisit the earlier problem:

```
My --> second --> personal --> project --> ?
```

Only this time, instead of predicting what word comes next, we
want to select a choose a word that sounds plausible. Well,
we found three good candidates by analyzing what other people
have written:

| Word     | Probability |
|----------|:-----------:|
| 'is'     |    50%      |
| 'was'    |    30%      |
| 'failed' |    20%      |

How do we choose between them? Well, we can be lazy and just
use the same probability distribution: we choose 'is' with 50%
probability, and so on. And that's it! The basic ideas behind Markov 
chain text generation. We can just keep tacking on words, repeating
the process every time.

Of course, we can do better. Most humans don't just write a single
word at a time: context is important. But that's much harder to model
compared to our fairly simple program. So how do we improve our
program without making it too complex?

##### **Improved Text Generation**

One idea is to increase the so-called order of the Markov chain.
Instead of only looking at the previous word, we can look at the previous
two, three, or n words, resulting in an n<sup>th</sup> order Markov
chain. I hope you haven't gotten tired of this example:

```
My --> second --> personal --> project --> ?
```

Let's say we want to use a 3<sup>rd</sup> order Markov chain. 
Instead of leafing through books looking for just the word 'project', we'd
look for the phrase 'second personal project', and record what
comes next. Intuitively, this does a better job of capturing the
context, and is still relatively easy to program.

The downside to increasing the order is data sparsity. Suppose
we get ambitious and try to program a 500<sup>th</sup> order
Markov chain. Well, the probability that you encounter the same
500-word sequence twice in a source text is exceedingly low. Instead
of a nicely distributed histogram, we'd probably end up with
a histogram like this:

```
1 |----|
  |    |   
  |    | 
  |    |
  |    |
0 ---------
   word
```

That's not very interesting. Our program is just going to spit out
the source text word for word, because that's the only example it has.
You can alleviate the problem by reading more source texts, but
even then, a 500<sup>th</sup> order Markov chain is probably infeasible.
So there's a tradeoff between novelty and accuracy.

##### **Adding Features**

Okay, time to transition to my existing Markov framework. These 
are some features I'd like to incorporate, in rough order of priority:

**GUI.** There should be a graphical user interface, which is more user-friendly.

**Data visualization.** There should be a way to see, given a seed string, 
what the conditional distribution for the next word is.

**Reading multiple files.** There should be a way to parse multiple files
for construction of a more accurate Markov chain.

**Pruning the data.** There should be a way to delete sequences that only 
appear once or twice in the model. Could help make the model more robust
against typos or weird one-off sentences.

**User interaction.** There should be an interactive textbox that shows
autocompletion suggestions from the Markov chain.

**Parsing source text.** Not sure about this one.  Stripping the source text 
of punctuation before feeding it into the parser may or may not help. I don't 
really want to figure out proper grammar generation for generated text, but 
having `this phrase` and `this phrase."` as separate representations in the 
model doesn't seem right.

**Web app.** There are way too many of these floating around the Internet,
but it would make a nice web development project. Might have to rewrite the
code in Javascript though.
