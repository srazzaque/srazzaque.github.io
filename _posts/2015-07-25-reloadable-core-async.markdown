---
layout: post
title:  "Reloadable core.async code"
date:   2015-07-25 11:42:00
categories: clojure
tags: clojure core.async concurrency
---

After being exposed to writing reloadable code on the front end using figwheel and Om, I started
wondering about how to write similarly on message-driven server applications.

The core.async library is fantastic, but it does have some gotchas when writing reloadable
code. Definition of reloadable here being:

- I can reload a given namespace such that it affects currently running code in a predictable manner
- I can stop/start the 'core' part of the program and a predictable and safe manner

Here are a few things I've started doing to maintain a reloadable workflow.

# Bring up channels in a function

Rather than this:

{% highlight clojure %}
(defonce incoming-msgs (chan))
(defonce outgoing-orders (chan))
;; Do stuff here
{% endhighlight %}

Prefer:

{% highlight clojure %}
(defn start
  []
  (let [incoming-msgs (chan)
        outgoing-orders (chan)]
     ;; Do stuff here
     ))
{% endhighlight %}

The reason for this is that, even if we gracefully do bring down all of our running go-blocks, there
are times when a message will remain in the channel even after stopping. It's best to just re-create
the channel as part of the startup routine to avoid this altogether.

# Prefer functions that take their dependant channels as arguments

Kind of an extension of the above - but rather than this:

{% highlight clojure %}
(defonce incoming-msgs (chan))

(defn process-msgs
  [process-fn]
  (go
    (loop []
      (process-fn (<! incoming-msgs))
      (recur))))
{% endhighlight %}

Prefer:

{% highlight clojure %}
(defn process-msgs
  [msg-chan process-fn]
  (go
    (loop []
      (process-fn (<! msg-chan))
      (recur))))
{% endhighlight %}

Other than the fact that it's good form to have your functions declare their dependencies as
arguments, it's far easier to test/debug.

# Handle receiving nil

Parking take's (<!) will receive 'nil' from closed channels. The go-block should terminate accordingly.

Rather than:

{% highlight clojure %}
(go
  (loop []
    (process-fn (<! incoming-msgs))
    (recur)))
{% endhighlight %}

Prefer:

{% highlight clojure %}
(go
  (loop []
    (let [msg (<! incoming-msgs)]
      (when msg
        (process-fn msg)
        (recur))))
  (println "I've finished!"))
{% endhighlight %}

A little more verbose, but the go-blocks listening to those channels will complete gracefully on the
channel closing.

# Stop channels

I like to define a 'stop channel' that gets exposed to me, so I can stop all running go-blocks easily:

{% highlight clojure %}
(defn start
  []
  (let [stop-chan (chan)
        stop-chan-m (mult stop-chan)
        incoming-msgs (chan)]
    (go
      (<! stop-chan)
      (close! incoming-msgs))
    (fetch-msgs (copy-chan stop-chan-m) incoming-msgs)
    (process-msgs (copy-chan stop-chan-m) incoming-msgs)
    stop-chan))
{% endhighlight %}

We return the stop-chan above so that the caller has a hook to stop the process(es). To invoke it,
we put any message onto the stop channel.

The 'copy-chan' is a simple method:

{% highlight clojure %}
(defn copy-chan
  [chm]
  (let [copy (chan)]
    (tap chm copy)
    copy))
{% endhighlight %}

And fetch-msgs is a little more involved, due to the complexities of bridging the gap between the
outside world and our app, plus remaining responsive to the stop channel:

{% highlight clojure %}
(defn fetch-msgs
  [stop-ch incoming-ch]
  (go
    (loop []
      (let [msg (transport/recv-msg {:maxWait 200 :queue "system/incoming"})]
        (when msg ;; msg is nil when no message to receive after maxWait
          (let [[_ ch] (alts! [[incoming-ch msg] stop-ch])]
            (condp = ch
              incoming-ch (recur)
              stop-ch :complete)))))
    (println "fetch-msgs stopping.")))
{% endhighlight %}
