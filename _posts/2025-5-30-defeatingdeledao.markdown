---
layout: post
title:  "Defeating Deledao"
date:   2025-5-30 10:30:15 -0600
categories: cybersecurity proxy
---

# We had a problem,

I could no longer browse CS:GO cases at school. I couldn't watch my YouTube, couldn't read my docs, and couldn't even play Minecraft.

Deledao is a school-focused solution designed to manage and monitor student Chromebooks. The software broadcasts the current tabs to a server, where some AI algorithm determines whether or not the page should be allowed. Every image is blurred; everything is visible.

After Deledao finishes showing your every move, your teacher can enforce browser blocks through a Chrome extension installed on the laptop.

It also ensured the blocking of anything related to LGBTQ issues, which I believe was a manual school filter.

My school spent thousands on this software, but I was able to bypass it within a few days. Now that I have safely graduated, I will show you how to do the same.

# How does it work?

Deledao sends REST API requests to a backend server periodically, which then receives a response indicating whether the request can be displayed. It scans all text, all images, and all video to ensure it is "safe."

Admins are provided with a local network IP address so that the Chromebook screen can be streamed efficiently to the admin's PC.

# Setting up a proxy server

> This will not be technical. I'm trying to make this as easy as possible to replicate, with no technical knowledge required.

With no server authentication, the solution should be obvious: all you must do is proxy out the responses to the backend, and you're done. You can also, as I figured out later, spoof fake requests through your proxy to change the computer that your screen is viewing; i.e., my screen appears to be normal, but it's my friend's in reality. (*Remember the local IP included in the request?*)

The specifics of what to do from here are simple: download **mitmproxy**. This tool serves as an HTTPS proxy that is easily scriptable, making it ideal for our use case.

Find a computer to run during school, download mitmproxy, and run the following command: `mitmproxy --set block_global=false --set intercept=deledao --set listen_port=8080`. This will allow for connections outside your local network and intercept all requests made to Deledao.

Next, port forward this local machine in your router config. There is no single way to do this, as all routers implement it slightly differently, so you may need to consult the documentation for your specific router. Once you get there, you need the ***LOCAL IP ADDRESS*** of your mitmproxy computer. Look this up as well; it's dependent on your OS. Once you know how to port forward and the local IP address, create a port forwarding rule to that local IP at port 8080. Some, if not all, routers allow you to select two different ports. For simplicity, put 8080 for both of these values.

> The port being 8080 is not required. I just chose it when I set mine up. If, for some reason, you need to use a different port, ensure you change the listen_port option in the above command.

# Setting up Chromebooks

Once you have a proxy server set up and running, determine your home's public IP address and remember the port number you are using.

The setup of a proxy may differ depending on your school's Chromebook policy. Mine would not allow a proxy config on the official network, but luckily, it would allow it on a VPN connection or a phone hotspot connection.

> If you are technical, set up an OpenVPN docker container. This means you can avoid using phone data. This, of course, also means your proxy IP will change back to the local address on the VPN. Alternatively, find a free server config online.

Next, navigate to mitm.it in your browser. This will let you download the root certificate for HTTPS communications. 

> Note: This will allow the host of mitmproxy to see all communications, even over HTTPS. Use with caution!

 This is useful for scripts that target specific parts of the requests to Deledao, but it can also cause breaches of privacy.

Look up the procedure for adding a new certificate authority to a Chromebook, as it varies by device and version.

# All done!

Now you're done! This method works as long as your server is active. I was fortunate to have my own server at home, which I use to host various things, allowing me to quickly open a port and keep it running constantly.