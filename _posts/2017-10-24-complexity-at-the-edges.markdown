---
layout: post
title:  "Complexity spikes at the edges of software design"
date:   2017-10-24 00:00:01
categories: software-development
tags: software-development
---

It is thrilling for programmers to design simple and elegant solutions to interesting problems. But
when we don't consider the "edges" as part of our design, we (or our customers) end up with
pain. Lots of it.

By the "edges", I don't necessarily mean system boundaries. I'm referring to the edges of a problem
space - quite often referred to as edge/boundary cases, non-functional requirements and practical
considerations. This is generally the point at which it starts to become an "uninteresting problem"
in a challenging/engineering sense.

Working at many banks, I've come across many proprietary applications - both in-house and
vendor-created. I'll discuss one of these applications, let's just call it SliceDB. SliceDB has a
running state that is quite nice and elegant - and I can imagine that the engineers got quite a buzz
from solving the problem in this manner.

It's essentially a system that allows the updating and querying of real-time trade data, which it
holds in memory for extremely fast query processing. The system is comprised of N different running
processes, each responsible for its own "slice" of time, keeping all data in memory - not too
dissimilar to sharding in many of today's NoSQL databases. Each slice will hold a particular
business day's trades - "T" for today's trades, "T-1" for yesterday's, etc. A query parser and
distributer transparently uses the appropriate slices for the query being served.

Let's ignore the fact that a lot of this today can be solved with a multitude of "big data" systems
off the shelf - this product was written quite a while back and well before the big data
explosion. I wanted to focus on the fact that the design and simplicity of this system quickly
breaks down at the edges.

What's the procedure to roll the dates? To persist the data from each slice so we don't lose it?
Bringing up/down slices? Deployment and configuration? And in the midst of performing all these
procedures, how can we easily reason about what state the system as a whole is currently in? These
sorts of questions uncover what I consider to be the edges of a design.

The answers to these questions in this particular product, as with so many proprietary enterprise
products I've used, is "run this hacked together script.". When there's trouble, well, you just
have to know what to do from last time. The design at the edges causes a massive spike in
complexity and annoyance that is incongruent with the simplicity and elegance of its running state
design. The flaky procedures for working with these 'daily matters' by far caused the most pain in
day to day work with the system. It requires an insane amount of internal product knowledge, which
you can basically only get from its original engineers/vendor.

Anyone that has worked in a large corporation is bound to have used or seen software like
this. Everything outside the scope of that zen state is never really talked about, considered,
mentioned in sales meetings - so it goes unchecked, and developers and support staff are left with
pain.

So the lesson to me is that great software design has simplicity and elegance all the way through to
its edges.
