---
layout: post
title:  "Writing simulators in core.async"
date:   2015-02-20 00:00:01
categories: clojure
tags: clojure core.async concurrency simulators
---

As developers, we take on the risk of delivering large and complex applications to our
clients. Often, contracts and the nature of the relationship places constraints on how we
develop. An example is having a customer who has asked you to build a system to be deployed within
their environment, but to develop it on your own turf. To do this, we need a precise understanding
of the integration between our application and the client's other systems.

To this end, simulators are a great way of (1) exploring integration requirements and (2) maximizing
the chance of our applications actually working in the client's environment. And after some
experimentation, core.async looks like a promising choice for building them. Let's explore why.

Note: some preliminary knowledge of clojure and core.async would be helpful before reading this post.

# Why simulate?

Let's say we're delivering a trading system to BigBank. One of the many things that our system
_won't_ be responsible for is pricing. We'd be pulling prices from BigBank's own pricing
engine. But, we don't have that pricing engine available to us for testing as it is proprietary
code.

There could be other reasons for not having a test system available - perhaps the client/exchange
doesn't have a test instance, or they're ridiculously expensive. I've once developed against a
system that was "production" during business hours and "test" during all other hours. Don't ever do
that.

But, despite system availability - simulators are a good idea if you're offsite or otherwise
disconnected from that environment and the team that maintains that other system. Simulation offers quite
a few benefits:

* It simplifies your application & infrastructure code by having just one code path. This is
  provided that your simulators are interacting with your app in the same way as production;
* It forces you to consider how the interfacing system(s) work and interact with your system and
  therefore the high-level design of all the interacting components; and
* You end up exploring interface requirements via code rather than documentation, which forces you
  to understand it more thoroughly.

But - simulators do add a code maintenance burden. They may grow into a large codebase that needs to
be maintained alongside your production code. This is why I think clojure and core.async may be a
good choice. Clojure code tends to be terser than Java (due to a huge reduction in line noise), and
deals very well with concurrency issues.

# Our requirements

Let's add some details to our earlier requirements. This is a highly stripped-down version of a
system I've worked with in real life, but let's assume that the expected interaction between a
pricing subscriber and a consumer is as follows:

We have a single pricing engine. Upon subscription, the engine will pump out prices for that
subscription for 30 seconds, then stop. If subscribers attempt to re-use a subscription ID, then a
message is logged and nothing will be published for that subscription.

# Simulator core

Our pricing simulator can be thought of as two main procedures:

1. Request receiver: Receive subscription requests, validate, and create subscriptions
2. Price publisher: Send prices to a subscriber for a given subscription

In terms of cardinality, we'd have a single running (1) procedure, and multiple (2)'s - one for each
open subscription. The first of these procedures is fairly straight-forward to express:

{% highlight clojure %}
(defn receive-price-requests
  [request-chan seen-ids receive-fn]
  (go
   (while true
     (let [request (<! request-chan)
           id (:subscription-id request)] 
       (if (valid-request? seen-ids request)
         (do
           (log/infof "Processing price request: %s" (pr-str request))
           (swap! seen-ids conj id)
           (receive-fn request))
         (log/warnf "Received an invalid request: %s" (pr-str request)))))))
{% endhighlight %}

Even a programmer who doesn't understand clojure would be able to understand what the code above is
doing. We continually loop in the go-block to receive, validate the request against seen-ids, and
call a function (receive-fn) using the request if it is valid.

Let's define something we can use for that receive-fn (we'll see how to tie it in further below):

{% highlight clojure %}
(defn create-subscription
  [request {:keys [subscription-period max-wait max-interval] :as opts}]
  {:pre [subscription-period max-wait max-interval]}
  (let [c (chan)
        end-time (+ (System/currentTimeMillis) subscription-period)]
    (log/infof "Creating subscription for: [%s] with options [%s]" (pr-str request) opts)
    (go
     (while (< (System/currentTimeMillis) end-time)
       (<! (timeout (rand max-interval)))
       (let [snapshot (random-snapshot request)]
         (alts! [[c snapshot]
                 (timeout max-wait)])))
     (close! c))
    c))
{% endhighlight %}

The go-block will continually loop while the subscription is not over, and publish a random snapshot
onto the prices channel. 

Within the loop, the first thing we do is park on retrieving a value from a timeout channel. This
will effectively 'suspend' the go block for a random period of time before continuing.

Then, we use the alts! macro to send a snapshot into the defined channel, but wait a maximum of
'max-wait' before looping again. The alts! macro (and its cousin alt!) allow you to specify one or
more 'channel operations' (either puts or takes), and whichever is ready first will proceed. In our
case, if the timeout channel returns something first, then we proceed without placing an item onto
our 'c' channel (and loop again if necessary).
  
Finally, we return the channel where the subscription's prices will be placed - the caller is free
to do whatever it pleases with the resulting channel.

We almost have a working simulator. All that remains is a means of...

# Hooking it up to the real world

To hook it up to the real world, we need to create a few more procedures:

1. A process to continually read messages from some messaging source and pump them into a channel
2. A process to continually read values from a channel and pump them into a messaging source
3. A serialization/deserialization mechanism (obviously aligned to what your application is to use.)

The above pieces could be considered infrastructure pieces. Once written, they could be used across
the code base for other simulators.

Suppose we had some transport-specific methods defined in the 'zmq' namespace. We can then hook
channels up to the transport quite easily:

{% highlight clojure %}
(defn- wire-to-zmq
  [request-chan prices-chan]
  (let [zmq-ctx (zmq/new-context)
        request-sock (zmq/subscriber zmq-ctx price-request-addr)
        prices-sock (zmq/publisher zmq-ctx price-addr)
        json-deserialize-fn #(json/read-str % :key-fn keyword)]
    (zmq/start-listener request-sock
                        (comp map->PriceStreamRequest json-deserialize-fn)
                        #(>!! request-chan %))
    (zmq/start-publisher prices-sock
                         json/write-str
                         #(<!! prices-chan))))
{% endhighlight %}


# Tying it all together

And at long last - we have our core simulator function which will tie everything together:

{% highlight clojure %}
(defn start-simulator
  []
  (log/info "Starting pricing engine simulator.")
  (let [request-chan (chan)
        prices-chan (chan)
        seen-ids (atom #{})
        subscription-opts {:subscription-period 30000
                           :max-wait 2000
                           :max-interval 500}]
    (receive-price-requests request-chan
                            seen-ids
                            (fn [request]
                              (let [subscription-chan (create-subscription request
                                                                           subscription-opts)]
                                (pipe subscription-chan prices-chan false))))
    (wire-to-zmq request-chan prices-chan)))
{% endhighlight %}

# Conclusions

Though I'm yet to use a core.async-backed system (simulator or otherwise) in a production setting,
from this little exploration I can conclude a few things:

* The CSP programming model allows you to break your problem up quite naturally
* Terseness and simplicity of functions should mean easier maintenance
* The amount of 'noise' vs Java's concurrency primitives (and Java in general) is incredibly low
* I've only touched the tip of what core.async is capable of doing. The library offers up a tonne of
  methods and primitive functions.

So, in short, I think I'll be playing around with core.async a lot more!