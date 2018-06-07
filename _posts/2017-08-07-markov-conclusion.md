---
layout: post
title: Markov Conclusion
date: 2017-08-07
description: A summary of the finished Markov Chain Text Generator program.
category: [markov]
---

Unfortunately I've been caught up in research and other things this summer, but after
two months, I've finally gotten around to completing this project. The main goal was
to familiarize myself with GUI programming in Java again, to prepare myself for an
upcoming semester of TA'ing CS 2112--so despite the remaining bugs I'm sure are 
lurking around, I'm ready to call it a day.

##### **Screenshot**

<img src="/assets/markov.png" align="center" width="100%">

##### **Features**

- GUI built with JavaFX and SceneBuilder
- Adjustable Markov chain order
- Ability to parse multiple text files and get data from all of them
- TAB autocompletion
- Most common next word statistics

##### **Usage**

The Markov.jar file in the Github repo contains everything needed to run the program. 
Downloading and double-clicking it should launch it, but if not, you can try typing

`java -jar Markov.jar`

from the terminal.

##### **Summary**

In theory, this program allows you to parse works by multiple authors, and then generate
text that sounds like a combination of all of them. For example, if I feed in 
*Pride and Prejudice* and *Huckleberry Finn*, I'll get text that seems to wobble in between
the two styles of writing:

> *I can’t imitate him; but husband’s going over there to do it but your own way; but when I 
> read considerable to Jim about kings and dukes and earls and such, and how your efforts 
> and donations from donors in such confusion!” “Oh! Jane,” cried Elizabeth, darting from her 
> beyond a monosyllable. Miss Darcy likely to know more of quickness than her sister. Mr. Collins 
> made his bow and do the other Dukes of Bilgewater was a big old-fashioned double log-house 
> before I came to town every day was coming.*

That said, your mileage may vary. It's sensitive to the size of the source texts: a larger 
work is going to dominate over a smaller one in terms of word weights.

I also never got around to automatically cleaning the input data, which means all of the punctuation
is still parsed, and phrases like "word", "word,", and "word:" are all counted separately
by the program.

##### **Design**

There was one major design choice I had to make. An issue arises when you type in a chain of
words that the Markov chain has never encountered before. From the back-end, you can either throw 
an error, or try to match a subset of the prefix. If you throw an error, then the front-end can't 
really display any information about what words come next. No statistics, no auto-completion. That's 
not very fun.

Instead, I had the chain match one word of the prefix at a time until it couldn't find the next word.
For example, suppose I had the phrase `one fish two fish`. First the program tries to match the word
`one`, and checks if the word `fish` ever comes next. If it does, then great. We just repeat the process.
Otherwise we terminate our search, and give the user the distribution of words that come after the
prefix `one`. In the edge case that the word `one` never appears in the source text, then we return 
the distribution of all possible first words in the source text.

I suppose this doesn't exactly follow the theory, but the change does make the program more interactive.
The user doesn't need to know if a specific sequence of words appears in the source text: they can
type whatever they want and auto-complete arbitrarily. 
