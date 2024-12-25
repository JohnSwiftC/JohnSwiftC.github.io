---
layout: post
title:  "Quick Networking for Windows"
date:   2024-12-25 12:00:00 -0600
categories: cybersecurity maldev networking
---

# Quick Start on Windows

Windows uses the same (more or less) socket API that Unix systems do, that is functions like `socket`, `bind`, `listen`, `send`, and many more. However, we must first initialize WinSock before we can use any of these functions.

{% highlight c %}
WSADATA WsaData; // Create the WsaData
WSAStartup(MAKEWORD(2, 2), &WsaData);
{% endhighlight %}
This must be done before any networking on Windows. It is also then good practice to call `WSACleanup()` after you are done.

# Server Networking, General

Client and server code looks more or less the same, except that a server must also use `bind`, `listen`, and then `accept`.

We will be going over the server in detail first. First, we need to quickly understand the `addrinfo` structure.

{% highlight c %}
typedef struct addrinfo {
  int             ai_flags;
  int             ai_family;
  int             ai_socktype;
  int             ai_protocol;
  size_t          ai_addrlen;
  char            *ai_canonname;
  struct sockaddr *ai_addr;
  struct addrinfo *ai_next;
} ADDRINFOA, *PADDRINFOA;
{% endhighlight %}

The `addrinfo` struct, as the name suggests, contain information about an address. Flags, family, socket type, protocol, and more.

In order to get a socket and bind, we need to get the `addrinfo` of the server (which is us, in this case.) Here is an example using `getaddrinfo`

{% highlight c %}
struct addrinfo hints, * servinfo;

memset(&hints, 0, sizeof hints);

hints.ai_family = AF_INET // ipv4
hints.ai_socktype = SOCK_STREAM; // tcp socket
hints.ai_flags = AI_PASSIVE; // we are the server, use own IP

getaddrinfo(NULL, "4000", &hints, &servinfo);
{% endhighlight %}

Here we see how we can use the `getaddrinfo` function. First, we create hints and * servinfo. Hints is an `addrinfo` struct that we use to configure how we will use `getaddrinfo`, and servinfo is a pointer that receives the output from `getaddrinfo`.

Now we can use our `addrinfo` to create a socket and start listening for connections.

{% highlight c %}

int sockfd; // Socket File Descripter, can be type SOCKET
int rv; // Return Value

sockfd = socket(servinfo->ai_family, servinfo->ai_socktype, servinfo->ai_protocol);

if(sockfd == -1) {
    return -1; // Error on socket.
}

rv = bind(sockfd, servinfo->ai_addr, servinfo->ai_addrlen);

if(rv == -1) {
    closesocket(sockfd);
    return -1; // Error on bind.
}

rv = listen(sockfd, 10);

if(rv == -1) {
    closesocket(sockfd);
    return -1; // Error on listen.
}

{% endhighlight %}

Take a second to think about what we are doing. We first get the `addrinfo` that represents us as a server. We then open a port and store the result as a socket file descriptor. Think of sockfd as a handle to the socket that we just opened, and we will need it for control over that socket.

Also notice that while we are listening for connections, we are not accepting any. This leads us into using the `accept` function.

{% highlight c %}

int clientsockfd;

struct sockaddr_storage clientAddr;
int sin_size = sizeof clientAddr;
char clientipv4[INET_ADDRSTRLEN];

clientsockfd = accept(sockfd, (struct sockaddr *)&clientAddr, &sin_size);
inet_ntop(AF_INET, &(((struct sockaddr_in*)&clientAddr)->sin_addr), clientipv4, INET_ADDRSTRLEN);

printf("%s%s\n", "Client IP: ", clientipv4);

{% endhighlight %}

Notice two things that I am doing that I will now explain in detail: using `accept`, and using structs like `sockaddr_storage` which I then use with casting to get the IP address of a connected client.

First, what does `accept` do? The `accept` function allows for a connecting client to make a connection, assuming that the socket is currently listening. It will return a new socket that is to be used for communicating with that client. This is where server design is important. Note that the accept function only accepts **one** client, and then moves right along. How do we deal with multiple clients? While I won't go in depth here, solutions like threading an accepting process are commonly used to deal with this.

Second, what is going on with the `sockaddr_storage` structure? Well, heres the deal. There exists a `sockaddr` struct in the `addrinfo` struct. It contains information about a socket. However, it is useless as a `sockaddr` struct. We need to cast it to `sockaddr_in` (IPv4) or `sockaddr_in6` (IPv6) to get data that we want out of it. `sockaddr_storage` is capable of holding both of these structs, so I use it for client information on accept. I then cast it to a `sockaddr_in` so that I can access data needed for the `inet_ntop` function, which takes a network address and converts it to the human readable IP notation. Structures are below.

{% highlight c %}
struct sockaddr {
   unsigned short    sa_family;    // address family, AF_xxx
   char              sa_data[14];  // 14 bytes of protocol address
};


struct sockaddr_in {
    short            sin_family;   // e.g. AF_INET, AF_INET6
    unsigned short   sin_port;     // e.g. htons(3490)
    struct in_addr   sin_addr;     // see struct in_addr, below
    char             sin_zero[8];  // zero this if you want to
};


struct sockaddr_in6 {
    u_int16_t       sin6_family;   // address family, AF_INET6
    u_int16_t       sin6_port;     // port number, Network Byte Order
    u_int32_t       sin6_flowinfo; // IPv6 flow information
    struct in6_addr sin6_addr;     // IPv6 address
    u_int32_t       sin6_scope_id; // Scope ID
};

struct sockaddr_storage {
    sa_family_t  ss_family;     // address family

    // all this is padding, implementation specific, ignore it:
    char      __ss_pad1[_SS_PAD1SIZE];
    int64_t   __ss_align;
    char      __ss_pad2[_SS_PAD2SIZE];
};

{% endhighlight %}

# Communicating, Send and Recv

Now established a connection as the server to a client, we can finally use the `send` and `recv` functions to pass data between the two. Look at the next example,

{% highlight c %}

int bytesRecv;
int bytesSent;

char messageBuffer[100];

bytesRecv = recv(clientsockfd, messageBuffer, sizeof messageBuffer, 0);

printf("Recieving %d bytes from client.\nMessage: %s", bytesRecv, messageBuf);
printf("Sending message back to client.");

bytesSent = send(clientsockfd, messageBuffer, sizeof messageBuffer, 0);

printf("Sent %d bytes back to the client.\n", bytesSent);

{% endhighlight %}

The code above recieves a client message, prints it, and then echos it back to the client. Notice how we use the `clientsockfd` we defined earlier on the `accept` function call. Look at the `send` and `recv` arguments as well. We pass the buffer, the size of the buffer, and then optional flags for both functions. Look at the Windows Documentation to find these optional flags.

# Client

The server is complete, and now we must make the client. The client, however, it much easier than the server. Look below.

{% highlight c %}

int sockfd;

struct hints, * p, * servinfo;

memset(&hints, 0, sizeof hints);

hints.ai_family = AF_INET;
hints.ai_socktype = SOCK_STREAM;

getaddrinfo("192.168.1.96", "4000", &hints, &servinfo);

for (p = servinfo; p != NULL; p = p->ai_next) {

    if((sockfd = socket(p->ai_family, p->ai_socktype, p->ai_protocol)) == -1) {
        continue;
    }

    if(connect(sockfd, p->ai_addr, p->ai_addrlen) != 0) {
        closesocket(sockfd);
        continue;
    }

    break;

}

if(p == NULL) {
    return -1; // Loop found no good sockets
}

char message[] = "Hello World!";
char buffer[100];
int bytesSent;
int bytesRecv;

bytesSent = send(sockfd, message, sizeof message, 0);
printf("Sent %d bytes to server.\n", bytesSent);
bytesRecv = recv(sockfd, buffer, sizeof buffer, 0);
printf("Received %d bytes from server.\nMessage: %s", bytesRecv, buffer);

{% endhighlight %}

Notice how we now specify an IP address in `getaddrinfo`. We also do not set the AI_PASSIVE flag, because we are connecting to a server. Notice the use of a loop here. The `addrinfo` structure acts like a linked list, with the `ai_next` field being a pointer to *another* `addrinfo` struct. We iterate through this list to find a valid socket that we can connect to the server on.

Then we send and recv from the server! Before you try this out yourself in Visual Studio or however you're building this on Windows (definitly just use Visual Studio!), look at some instructions for building on Windows.

# Building, Includes, Annoying Stuff

Building for Windows is both easier and harder than Linux. First, we do not need to include the thousand headers needed for networking in Linux, we only need `#include <winsock2.h>`. However we aren't done yet. If we plan on using Windows API later, we must put `#define WIN32_LEAN_AND_MEAN` at the top of our file. If we are building on Visual Studio as recommended, we also need to include `#pragma comment(lib, "Ws2_32.lib")` to get the proper functions.

There it is! you have the most basic and fastest guide to networking! For a much more in depth but also longer guide, go look at my personal favorite [Beej's Networking Guide](https://beej.us/guide/bgnet/html/split/). It covers in detail a lot more than I did here, including history, IPv6, UDP, broadcasting, multicasting, server techniques, and more! 💜