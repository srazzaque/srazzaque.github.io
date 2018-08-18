---
layout: post
title:  "Monitoring stuff through Dropbox"
date:   2018-08-18 00:00:01
categories: software-development
tags: software-development dropbox
---

TLDR: write command output to a text file in your Dropbox folder, and look at it from any device.

If you've got the Dropbox Linux daemon running, it's super-easy to monitor stuff on a Linux machine
from any phone or tablet. The "traditional" way to do this would be to use an email (sendmail), but
this is far more lightweight, realtime and flexible if you already happen to have Dropbox set up.

{% highlight bash %}
[srazzaque@srv ~]$ for num in 0 1 2 3 4 5 6 7 8 9
> do
> echo $num > ~/Dropbox/a_file.txt
> sleep 5
> done
{% endhighlight %}

Perhaps more common would be:

{% highlight bash %}
[srazzaque@srv ~]$ long-running-cmd && echo "done" > ~/Dropbox/command-done.txt
{% endhighlight %}

And that's basically it - you can view the files on any device that supports Dropbox.
