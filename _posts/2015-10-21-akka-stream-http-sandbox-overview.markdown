---
layout: post
title:  "Akka Stream, Akka HTTP"
date:   2015-10-21 19:00:00
image:  /assets/article_images/DSCF0808.JPG
---

I was playing around with Akka Stream and Akka HTTP and thought I could come up with a sample project which is a bit more complex than a Hello World.


### The problem

Let the imaginary problem be this: we want to send some sort of personalised messages to someone on twitter, facebook, and email.

1. Input: recipient ID
2. look up the recipient's name
3. generate messages for a predefined list of platforms (facebook, twitter, email); there might be more than 1 message per platform (we want to spam :) or to overcome message length limitations)
4. upload the messages in batches; different batches per platform of course

Let step 2 and 3 happen in different apps.


### What to expect

I am going to go through some topics, such as

- A simple Akka HTTP service.
- How to turn an Akka HTTP endpoint (with routing) to an Akka Stream `Source`. [here]({% post_url 2015-10-25-bind-akka-route-to-stream %})
- How to call HTTP services with Akka HTTP client, and _handle errors_ without breaking the stream.
- How to fan out to multiple `Sink`s.

I also plan to do some dockerisation and performance measurement around this in the near future.

The code is on [GitHub](https://github.com/szmg/akka-stream-http-sandbox).
