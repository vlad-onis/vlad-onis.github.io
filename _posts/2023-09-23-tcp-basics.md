---
title: TCP basics
date: 2023-09-23 13:00:00
categories: [Rust, Web Servers]
tags: [rust]
---

This blog post explores the basics of the TCP (Transmission Control Protocol). We'll look into some nitty gritty details of TCP and then we'll implement a very naive first version of our soon to be web server.


## Introduction

TCP is a communication protocol that ensures reliable (meaning the sender is notified about the receiving of their packets), ordered and error-free delivery of data. TCP is a connection oriented protocol, which means that before any communication could happen the parties have to establish a connection and perform a handshake.

### Connection

How do computers connect with each other you might ask? Well, TCP uses what is known as a 3 way handshake. The client is sending a message containing a SYN flag (which basically tells the server "let's synchronize"), a sequence number ISN (randomly chosen) and other control information.

If the server is able to accept this connection, it will chose its own ISN and send it back to the client while also incrementing client's ISN to let the client know that his message was received correctly. 

Upon receiving the server's message, the client acknowledges by also incrementing the server's ISN. These are the 3 steps that will establish a connection between the 2 parties.

To close a connection gracefully TCP performs what is known as a 4 way handshake. It is pretty similar to the 3 way handshake above but the parties send a FIN flag instead of SYN to signal they want to end the connection. Then the 4 way handshake allows final messages if there was still data to be sent by one of them. And then the acknowledgement of the connection closing will begin. This part also involves leaving the connection open a bit more (also known as TIME WAIT) just in case.

![TCP_3Handshake](/assets/img/tcp_basics/tcp_3handshake.png)

### Sockets

We keep hearing that when devices communicate over a network they use sockets 🧐. A socket is just an abstraction similar to wall sockets, you can plug something in them to create a bidirectional communication channel. We identify sockets by the combination of (IP_Address, Port_Number). The IP helps us identify the device and the port identifies the application we're trying to talk to. Sockets are generally managed by the operating system which ensures error handling (when something is happening with the communication), multiplexing (managing multiple sockets), routing of data to the correct socket and so on. The O.S. provides a socket API that your app can use to create and manage sockets.

Usually sockets are abstracted away by the library makers. Just for the sake of example, in Rust when you read a TCP message from a TCPStream object, you are actually doing a syscall do receive data from the underlying socket.

## Enough talking

**Server code**
```rust
fn greet(mut stream: TcpStream) {
    // // Set up reading
    // let _ = stream.set_read_timeout(Some(Duration::from_micros(10)));
    // let mut buf: Vec<u8> = Vec::new();
    // let _ = stream.read_to_end(&mut buf);
    
    let _ = stream.shutdown(std::net::Shutdown::Read);

    let response = "HTTP/1.1 200 OK\r\nConnection: Closed\r\n\r\n";
    let _ = stream.write_all(response.as_bytes());
}

fn main() -> Result<()> {

    let listener = TcpListener::bind("127.0.0.1:3000")?;

    for stream in listener.incoming() {
        println!("####New connection received");
        if let Ok(s) = stream {
            greet(s);
        }
    }

    Ok(())
}
```

**Client**
```bash
curl -vvv 127.0.0.1:3000
```

Notice that when the server creates the **listener** variable it passes an IP and Port number which should remind you about the definition of a Socket. Let's check bind's signature.
See how the bind function receives something that can be converted to a SocketAddr and returns a TcpListener or an io::Error
```rust
#[stable(feature = "rust1", since = "1.0.0")]
    pub fn bind<A: ToSocketAddrs>(addr: A) -> io::Result<TcpListener> {
```

Our listener then loops forever, waiting for connections. When a connection is accepted by the listener the greet function is called which simply write a "Connection Closed" message (in HTTP format) back to the client. 

The commented lines or the stream.shutdown() call are worth mentioning. If we don't shutdown the stream's reading end and we also don't read from the stream, at the end of the function when the stream will be dropped a RST packet will be sent back to the client. This is by design in TCP, although not mentioned explicitly in the HTTP RFC (https://datatracker.ietf.org/doc/html/rfc7230), request-response communication assumes the request is being parsed instead of being discarded. Try it yourself to see what cURL has to say about this

## Outro

Thank you for exploring together with me the wonders of this very naive and sync TCP implementation.

**<span style="color:green">It was fun, see you on the next one!</span>** 🚀🦀




