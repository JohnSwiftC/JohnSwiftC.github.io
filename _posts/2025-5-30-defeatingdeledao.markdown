---
layout: post
title:  "Defeating Deledao"
date:   2025-5-30 10:30:15 -0600
categories: cybersecurity proxy
---

# We had a problem,

I could no longer browse CS:GO cases at school. I couldn't watch my YouTube, couldn't read my docs, and couldn't even play Minecraft.

Deledao is a school focus solution dedicated to controlling and monitoring student Chromebooks. The software broadcasts current tabs to a server where some AI slop determines whether or not the page should be allowed. Every image is blurred, everything can be seen.

After Deledao finishes showing your every move, your teacher is able to enforce browser blocks through a chrome extension installed on the laptop.

It also ensured to block anything related to LGBTQ issues, which I believe was a manual school filter.

My school spent thousands on this software, but I had bypassed myself and my friends within a few days. Now that I am safely graduated, I will show you how to as well.

# How does it work?

Deledao sends REST API requests to a backend server periodically, which is then given a response determining whether or not the request can be shown. It scans all text, all images, and all video to ensure it is "safe".

Admins are fed a local network IP address so that the chromebooks screen can be streams efficiently to the admin's PC.

# Setting up a proxy server

> This will not be technical. I'm trying to make this as easy as possible to replicate with no technical knowledge.

With no server authentication the solution should look obvious, all you must do is proxy out the responses to the backend and you're golden. You can also, as I figured out later, spoof fake requests through your proxy to change the computer that your screen is viewing, ie my screen appears to be normal, but it my friend's in reality. (*Remember the local IP included in the request?*)

The specifics of what to do from here are simple: download **mitmproxy**. This tool is used as an HTTPS proxy that is easily scriptable which makes it great for our use case.

Find a computer to run during school, download mitmproxy, and run the following command: `mitmproxy --set block_global=false --set intercept=deledao --set listen_port=8080`. This will allow for connections outside your local network and intercept all requests made to Deledao.

Next, port forward this local machine in your router config. There is no one way to do this as all routers do it slightly differently, so you may need to look this up. Once you get there, you need the ***LOCAL IP ADDRESS*** of your mitmproxy computer. Look this up aswell, its dependant on your OS. Once you know how to port forward and the local IP address, create a port forwarding rule to that local IP at port 8080. Some, if not all, routers allow you to select two different ports. For simplicity, put 8080 for both of these values.

> The port being 8080 is not required, I just chose it when I set mine up. If for some reason you need to use a different port, ensure you change the listen_port option in the above command.

# Setting up Chromebooks

Once you have a proxy server set up and running, determine your homes public IP address, and remember the port number you are using.

Now, the setup of a proxy may differ based on your school's chromebook policy. Mine would not allow a proxy config on the official network, but luckily it would allow it on a VPN connection or a phone hotspot connection.

> If you are technical, setup an OpenVPN docker container. This means you may be able to avoid using phone data. This of course also means your proxy IP will change back to the local address on the VPN. Alternatively, find a free server config online.

Next, nagivate to mitm.it in your browser. This will let you download the root certificate for HTTPS communications. Small note, this will allow the host of mitmproxy to see all communications even over HTTPS. This is useful for scripts that target specific parts of the requests to Deledao, but also can cause breaches of privacy.

Look up the procedure for adding a new certificate authority to a Chromebook as it varies by device and version.

# All done!

Now you're done! This method works as long as your server is active. I was lucky to have my own server at my house that I use to host other things, so I was just able to quickly open a port and keep it running constantly.