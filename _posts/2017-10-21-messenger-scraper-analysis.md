---
layout: post
title: Messenger Scraper Analysis
date: 2017-10-21
description: More thoughts on the Messenger scraper, including Java's functional programming features, GUI design, and optimization.
category: [project, scraper]
---

Here, I explain more ideas behind the finished Messenger scraper GUI, including Java's functional programming features, GUI design, and optimization.

##### **Background**

[Here's the introductory post, if you haven't read it.]({% post_url 2017-10-03-messenger-scraper-introduction %})

I switched languages from Ruby to Java primarily to target a larger audience: people
are far more likely to use a GUI-based Java program than a Ruby command-line script.

##### **Overview**

The parsing code closely resembles the original Ruby script: it's also split
into three classes, corresponding to the major XML elements in the original data.

```
src
└── main
    ├── java
    │   ├── gui
    │   │   ├── FacebookMessageScraper.java
    │   │   ├── Filter.java
    │   │   ├── Sort.java
    │   │   └── ThreadButton.java
    │   ├── parse
    │   │   ├── Message.java
    │   │   ├── Scraper.java
    │   │   └── Thread.java
    │   └── util
    │       └── Paginate.java
    └── resources
        └── gui.fxml

```

The most time-consuming part of this project was actually just writing the GUI code: trying
to make it intuitive and performant.

##### **Parsing**

I drew a lot of inspiration from my Ruby scripts, and implemented most of the parsing via
recursive constructors. For example, the thread constructor:

```java
public Thread(Element e, String user) {
	this.messages = new ArrayList<>();
	e.select(".message").forEach(message -> messages.add(new Message(message)));
	Collections.reverse(messages);
	this.people = new HashSet<>(e.select(".user").eachText());
	this.people.remove(user);
	this.ID = new HashSet<>(Arrays.asList(e.ownText().split(",[ ]*")));
}
```

Line by line, this Thread constructor:

1. Initializes an ArrayList to hold its messages
2. Selects XML elements with class 'message', routes them through the Message constructor, appends them to list
3. Reverses the order of messages (Facebook stores them in reverse chronological order)
4. Extracts the names of unique users by hashing all names that appear in the thread
5. Removes the client's name from the thread's user set
6. Assigns the thread an ID, which is the set of all Facebook IDs involved in the thread

Function chaining makes each of these operations easy to write (albeit at the cost
of some reduced readability).

Here's an interesting problem I ran into: it appears that
Facebook automatically splits up any threads longer than 10,000 messages into multiple threads.
So here's the problem statement: given a list of Thread objects, 
return a new list where all Threads with the same ID have their messages merged in 
chronological order. This is the solution I settled on:

```java
private void collapseThreads() {
	ArrayList<Thread> collapsed = new ArrayList<>();
	Set<Set<String>> unique = new HashSet<>();
	
	threads.forEach(thread -> unique.add(thread.getID()));
	unique.forEach(id -> {
		collapsed.add(
			threads.stream()
			.filter(t -> t.getID().equals(id))
			.sorted((a, b) -> a.getStartTime().compareTo(b.getStartTime()))
			.reduce((a, b) -> a.add(b))
			.get()
		);
	});
	threads = collapsed;
}
```

You can see that I'm making extensive use of Java 8's functional features, in the form of
Streams and lambda functions. The algorithm is pretty simple:

1. Identify all unique IDs by hashing each thread's ID
2. Iterate through all unique IDs
	- Find all threads corresponding to this ID
	- Sort them by start time
	- Add them together in order (logic is in the Thread class)
	- Append result to final list of collapsed threads
3. Return collapsed list

If you think of a faster or more elegant way to do this, let me know!

##### **GUI**

I wanted to make the GUI as familiar as possible, so I opted for a Messenger-style layout, with
different threads listed on the left, and the conversation on the right.

<img src="/assets/scraper.png" align="center" width="100%">

- Filter out group or private threads
- Sort by date, conversation length, or participant name
- Search for a specific name
- Save conversations as a text file
- See how many messages you've exchanged with someone
- Read old messages without lag

There were two main design decisions that I thought were noteworthy. 

##### Sort (and Filter)

The first was the implementation of filter and sort. Given their names, 
you might suspect that I used them with the
corresponding functional methods (Stream.filter and Stream.sort), and you'd be right. Here's
roughly what the code looks like:

```java
scraper.getThreads()
	.stream()
	.filter(f.getPredicate())
	.sorted(s.getComparator())
```

Let's look at (part of) Sort:

```java
public enum Sort {
	
	EARLY("Earliest First", (a, b) -> {
		return a.getStartTime().compareTo(b.getStartTime());
	}), 
	
	LONG("Longest First", (a, b) -> {
		return b.getMessages().size() - a.getMessages().size();
	});
	
	private String str;
	private Comparator<Thread> cmp;
	
	private Sort(String str, Comparator<Thread> cmp) {
		this.str = str;
		this.cmp = cmp;
	}
	
	public Comparator<Thread> getComparator() {
		return cmp;
	}
	
	@Override
	public String toString() {
		return str;
	}
}
```

Each Sort enum has two properties: a display String, and a Comparator for use
with the sort method. [(This is the documentation for Java 8's comparator.)](https://docs.oracle.com/javase/8/docs/api/java/util/Comparator.html)

This makes Sort and Filter extremely extensible. It only takes three lines
of code in this file to add a new type, and the rest of the GUI code just works.
For example, to add alphabetical sort:

```java
ALPHABETICAL("Alphabetical", (a, b) -> {
	return a.getPeople().compareTo(b.getPeople());
}),
```

Contrast this with a solution without lambdas:

```java
class AlphabetSort implements Comparator<Thread> {
	@Override
	public int compare(Thread a, Thread b) {
		return a.getPeople().compareTo(b.getPeople());
	}
}

ALPHABETICAL("Alphabetical", new AlphabetSort());
```

There's a lot more boilerplate code here that distracts from the
main sorting function. The lambda solution is much more elegant.

##### Pagination

The other design decision actually stemmed from performance issues. I was
initially trying to display entire threads in a single String object. After
a thread reached around 20,000 lines of text, the program started to lag quite a bit when
displaying the conversation. To solve the problem, I implemented a simple
Paginate class, which takes in some List, and returns parts of it at a time.

I noticed before writing Paginate that the same class would be useful
for paging through thread buttons as well. Although the buttons weren't
lagging nearly as much as the conversation text, there was still a massive
scrollbar to display all thread buttons at once. So I made Paginate a
generic class, to handle both thread buttons and messages at once.

The end result: lag-free viewing of conversations and thread buttons.