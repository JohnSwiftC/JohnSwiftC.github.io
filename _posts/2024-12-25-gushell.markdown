---
layout: post
title:  "GuShell"
date:   2024-12-25 10:30:15 -0600
categories: cybersecurity reverseshell
---

# Introduction to GuShell

My most recent project which is still receiving updates is [GuShell](https://github.com/JohnSwiftC/GuShell), a nice and simple reverse shell built for Windows entirely in C.

GuShell, taking inspiration from popular Red Team tools like Meterpreter, *a much better tool than mine*, is a client which gives an attacker new capabilities outside of a normal reverse shell through calling Window's API functions.

What did I learn in this project? A lot, actually. Building in C is always a fun experience, and it forced me to relearn a bunch of networking. Windows is also always a beast to work with, especially when we want to use some features in *unintended* ways... 😓

# Project Highlights

Some newer features, especially the more "malicious" ones, pushed me to learn a lot about operating systems and Windows exploitation. The big one is the client's ability to resolve API from the **PEB**, or the *Process Environment Block*. This feature is seen in a lot of malware but defeats many antiviruses as it is nearly impossible for a static analyzer to see all the evil functions you're calling 👻.

It does some other great stuff too, like moving itself into AppData (maybe more options soon?), and like I said earlier putting itself into the `Run` directory within the registry. Like other reverse shells out in the ether, it also allows the attacker to spawn shell processes and communicate with them! Definitly a nasty combination.

# Big Warning

This is definitly something I need to say for this one! GuShell, while being developed from an educational standpoint, is definitely still dangerous! If you are going to try out my project, ensure that you have permission on all machines involved.

Also, lets be real here. If you use this on a target that you ***are not supposed to***, you will 100% get caught. It literally connects back to your IP address! Blah blah *maybe* you could setup the listener on a C2 server, but there are better ways that don't involve my project.