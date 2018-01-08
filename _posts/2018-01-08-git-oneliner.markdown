---
layout: post
title:  "Git one-liners"
date:   2017-09-20 00:00:01
categories: software-development
tags: software-development git
---

A cute little git command to quickly see where the activity has been when exploring a large
codebase:

{% highlight bash %}
$ for d in $(find . -mindepth 1 -maxdepth 1 -type d | grep -v .git)
> do
> echo -n "$d,"
> git log --oneline --since=30.days $d | wc -l
> done
{% endhighlight %}

This will give you output like:

{% highlight bash %}
./dir1,384
./dir2,32
./dir3,532
{% endhighlight %}

From this point its trivial to visualise it as a pie-chart or some other mechanism.

