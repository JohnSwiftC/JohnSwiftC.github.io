---
layout: post
title: "Go Make a Rust Crate"
date: 2025-2-20 12:00.00 -0600
categories: programming
---

Go use my Rust crate! I made one recently to help myself out in some of my projects that act like an HTTP server. [Lazyhttp](https://crates.io/crates/lazyhttp) has functions to read streams and other such activity. To my suprise, shortly after posting, I quickly broke the 100 download mark. Then came the 200, 300, and so on. Now, as of me writing, it has broken 1000 downloads! Pretty good for something that I never intended for other people to be using.

Why should you make your own crate? It's fun! Think about a hard problem you've faced recently, maybe some lacking functionality or something that you just _hate_ implementing, maybe stop being a bum and write it yourself.

Big word of warning though, don't be dumb like I was. I made some disgusting implementation of a request object that just completely sucked (and in the process ignored the real and very popular http library). Fixed that in version 2.0.0, ideally I should have never done it. Take your time, write tests, don't jump the gun, _spellcheck_.

That's it. I wrote this to pad out the blog while I work on my botnet written in Rust.

> How far can I push these projects before I get in trouble?
