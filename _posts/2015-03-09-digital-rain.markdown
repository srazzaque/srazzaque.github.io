---
layout: post
title:  "Digital rain in famo.us/cljs"
date:   2015-03-09 21:30
categories: cljs
tags: clojure clojurescript famo.us cljs
---

A [colleague of mine](https://twitter.com/matt_d_rat) showed off some
really cool stuff with famo.us the other day. Though I'm yet to be
convinced as to how code in famo.us might scale to a huge application,
I couldn't resist the temptation to play around with it because it was
so freakin cool!

I decided to write something that would give its performance a bit of
a test - so the Matrix's "Digital Rain" seemed like an ideal
candidate.

The live demo is here: [Digital Rain](/digital-rain/index.html)

Code is here: [Github](https://github.com/srazzaque/digital-rain), and
the main piece of code is here
[matrix.cljs](https://github.com/srazzaque/digital-rain/blob/master/src-cljs/famous_cljs/matrix.cljs).

Obviously can use quite a few touch-ups and optimizations - for
example, currently the size is fixed. But I whipped it up fairly
quickly just to get an idea of how to use famo.us. I think
cljs/core.async/famo.us could potentially be a lethal combination for
certain classes of application.

One interesting thing to notice is each column has a completely
independent state. We wait a random amount of time between inserting
each character, and then wait an additional random amount of time
before starting to timeout-fade the character.




