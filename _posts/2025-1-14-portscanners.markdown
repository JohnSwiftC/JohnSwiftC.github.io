---
layout: post
title:  "Catching Port Scanners (going for my own Wi-Fi!)"
date:   2025-1-14 10:30:15 -0600
categories: cybersecurity reverseshell
---

# What am I doing?

Well, I recently started the process of learning (*relearningish*) Rust! In doing that, I elected to make some tools to flex my wings and to get used to the difficulty of a language like Rust.

So, I made [EasyPot](https://github.com/JohnSwiftC/easypot)! It's a neat little tool that logs information about incoming data on TCP ports. Like many of you are aware, malicious actors enlist many different tools to find cracks in router configuration (like open ports!) to then exploit.

This tool, like a few others, serves as a *honeypot*, or it captures and logs these malicious attempts on open ports.

# So, what did I find?

I had never really looked into stuff like this and I was not entirely sure what to expect. During my development of GuShell, I did notice every now and then that my listener would pick up rogue requests to port 3000, so I was expecting *something...*

I got a lot more than I expected, and logged **hundreds** of requests scanning my network for vulnerable services on open ports. One of my favorite attackers scanned port 666 tens of times for different bad services ~

{% highlight json %}
Port: 666 Remote IP: 45.79.114.123:44316
[
    "GET / HTTP/1.0",
]

Port: 666 Remote IP: 45.79.114.123:44318
[
    "OPTIONS / HTTP/1.0",
]

Port: 666 Remote IP: 45.79.114.123:44326
[
    "OPTIONS / RTSP/1.0",
]

Port: 666 Remote IP: 45.79.114.123:44336
[
    "Bad data, likely a port scanner.",
]

Port: 666 Remote IP: 45.79.114.123:42850
[
    "Bad data, likely a port scanner.",
]

Port: 666 Remote IP: 45.79.114.123:42858
[
    "Bad data, likely a port scanner.",
]

Port: 666 Remote IP: 45.79.114.123:42860
[
    "HELP",
    "Bad data, likely a port scanner.",
]
{% endhighlight %}

As you can see, this guy was going for it! FYI, the "bad data" line isn't actually bad data, it is the standin I have Rust using when it can't parse the data into some sort of UTF-8 string. In other words, it is just targeting services with different encodings (maybe).

# Conclusion

There were a shocking amount of these requests, having logged over 1000! Moral of the story is to ensure that your router does not have random machines port forwarded: keep everything closed!

In the future, I think that I will send fake version responses back to capture the method of attack being used.

(P.S.) You can report these IP addresses! Almost all of them are widely known in some database for being malicious actors, with many others reporting them for filtering purposes.