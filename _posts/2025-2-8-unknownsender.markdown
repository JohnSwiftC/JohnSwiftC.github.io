---
layout: post
title:  "Making the Unknown Sender Project"
date:   2025-2-8 12:00.00 -0600
categories: rust
---

# The Unknown Sender Project

Recently, following some introductory Rust programs I made (go look at my post from before!), I decided to dive headfirst into some more difficult applications. After careful deliberation I decided to make a webapp that allows users to send cute messages to random people and in theory be nice to eachother.

After thinking about it some more, and by more I mean about 5 minutes, I quickly realized how *horrible* of an idea that was! Someone could send something probably pretty bad, completely anonymously, to someone they don't even know. I obviously still built it. [Unknown Sender Project](https://unknownsenderproject.app)

> It might be in my own best interest to add some moderation to messages...

I actually learned a lot from this project.

# So, what did I learn?

I *immediately* chose to avoid almost any real HTTP framework or external crates in the spirit of making Rust as hard as possible, but also to really solidify my base in the language.

Turns out, that makes everything a **lot** harder! To avoid a month of pure wheel reinventing, I did unfortunately choose to use a *few* external crates to get the job done. But don't worry! I only used three crates, and any Rustacean can hopefully see where I'm coming from in choosing what I chose: Tokio, a Postgres API, and the Twilio API.

Tokio is an obvious choice, and I am not even close to smart enough to replicate an asynchronous runtime myself. Postgres was used because I wanted something quick and easy to dodge a good amount of boilerplate I deemed unneccesary, and Twilio because that's how I elected to send messages.

With all the excuses out of the way, I did learn a good deal about Rust but also server design. Using multiple async tasks to handle connections, the classic Arc\<Mutex\>, worker threads to clean TTL'ed structs, filtering IP addresses, manually parsing HTTP requests, ***doing SSL***, and probably abusing futures.

> Also, I know that with Nginx, doing SSL on the backend is not neccesary. I did it for the love of the game.

# How is Rust?

I do think that I like Rust, or that I would like to like Rust, but it does irk me in some ways. The most greivous thing I encountered were insanely long and convoluted types (like Arc\<Mutex\<Vec\<UnverifiedMessage\>\>\>), which make writing async code more time consuming and tedious.

My next problem is almost definitely a skill issue, and that is casting data through 4 different functions to reach the target user of that data. This is just an architecture problem that I generally dig myself into as I make abstractions too quickly, but making sure that my 23 different function calls and calls to those functions making the functions calls in main are all consistent is not a fun time.

> This is pretty much a me problem. Rust doesn't even change anything with that, I would do the same thing to myself in C!

However, everything else is super duper *clean*. I love knowing that my code will almost always do exactly what I want it to do as long as it successfully compiles. The compiler walks me through things and it can be creatively used to determine some things about your code. Also, this might be one of the fastest things I have ever written.

# Frontend

Nothing special. I used React. Kind of a jackhammer to a nail kind of situation but whatever.

# Deploying

Fun fact, I HATE THE CLOUD! That might be an overstatement, but I do infact dislike hosting services. Seriosuly, some of these services are basically robbing me with what I could do myself for free.

So that's exactly what I did: everything myself and for free! I feel like people make this too complicated, but it really is simple, as long as you dedicate yourself to learning a few tools for about a day.

1. Docker
2. Nginx

That is literally all you need. I personally use Docker Compose to host both my frontend and backend, with an image for each.

> Quick tip for beginners, PLEASE multi-stage your builds. It shreds down the size of your images. A Rust image with only one stage is about 4 GB, while that same program multistaged to a proper environment is only around 25 MB.

Nginx allows for load balancing, routing, reverse proxying, easy HTTPS, pretty much easy everything. There's a reason everyone uses it.

Docker allows me to deploy anywhere, scale easily, and adds a layer of security to anything that I deploy.

Finally, I just used an old laptop as my server. Definitely works!