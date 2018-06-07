---
layout: post
title: Messenger Scraper Introduction
date: 2017-10-03
description: Explains some of the ideas behind the Messenger scraper.
category: [scraper]
---

This program was written to address a few issues with Facebook Messenger. It's hard to view, search, and count old messages, and Messenger doesn't provide a way to export chats to a more easily viewable format. This was my prototyping process for a solution.

##### **Background**

Facebook allows you to download a copy of your data, which includes all of your old messages.
I was interested in seeing how many messages I'd exchanged with different friends, so I decided
to parse through said data, which comes in a messy .xml format.

##### **Prototyping**

I used a lightweight scripting language--Ruby--to get a basic parser up and running. 
Using [Nokogiri](https://github.com/sparklemotion/nokogiri) and Ruby's functional features, 
it only took an afternoon. Here's what I discovered along the way:

##### Data Format

The raw data is divided up into threads, which are further subdivided into messages:

```html
<div class="thread">
	ID0@facebook.com, ID1@facebook.com
	<div class="message">
		<div class="message_header">
			<span class="user">
				Name		
			</span>
			<span class="meta">
				Date
			</span>
		</div>
	</div>
	<p>Message text</p>
</div>
```

- The thread tag contains the internal ID number of the users, but not their names.

This means that, unless you query the Facebook API for the name corresponding to the
ID number, you can't get the names of people in the conversation if they didn't actually
send a message.

- Each message records the name of the person who sent it, and the time it was sent.

The time is formatted like this: "Friday, July 19, 2013 at 6:02pm EDT", which means
timestamps are only accurate to the minute.

- The actual message text is not inside the message div.

This is a little annoying.

##### Implementation

Ruby's function chaining and syntax made writing parsing code a lot more fun. For example,
this line extracts all of the message divs and converts each one into a Message object:

```ruby
data.css('.message').map { |message| Message.new(message) }
```

##### Audience

After I finished my prototype, I showed it to a few apartmentmates, who displayed some interest
in trying it for themselves. This is when I realized that a) Ruby isn't exactly a household
programming language, and b) command-line programs aren't very user-friendly.

#### **Next Steps**

The working prototype was neat enough that I wanted to bring it to a wider audience. That meant
two things: a GUI, and a more ubiquitous programming language. Java was a natural choice, given my
extensive Java and Javafx experience from last fall.
