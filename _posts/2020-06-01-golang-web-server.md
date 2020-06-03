---
layout: post
title: Building A Basic Web Server in Golang From Scratch
excerpt_separator:  <!--more-->
---

HTTP web servers are everywhere. They run everything from our search engines to our social media websites. Have you ever wondered how an HTTP web server works? This post dives a bit deeper into how to build your own web server.

To understand web servers, you first need to understand the basics of computer networking for the Internet. The Internet is built upon layers of abstraction in something called the Internet protocol stack. The five layers are the application layer, transport layer, network layer, link layer, and physical layer. We are primarily concerned with the application layer (HTTP) and transport layer (TCP).

HTTP stands for Hyper Text Transfer Protocol which is a protocol that web servers and clients use to communicate with each other. You can read about the HTTP protocol definition in the IETF (Internet Engineering Task Force) RFCs (Request for Comments). HTTP gives a common standard for applications to talk to each other.

HTTP runs on top of a transport layer called TCP (and UDP in some cases, but we will mostly concern ourselves with TCP). TCP is a connection-oriented protocol that enables reliable delivery of network packets. TCP runs on top of the network layer on a protocol known as IP (Internet protocol). It is important to note that IP is unreliable, meaning that a packet sent to this protocol won’t necessarily be sent to the receiver. TCP implements reliable delivery through retries.

We will first build a web server that runs on top of the TCP protocol, however, it won’t implement the HTTP protocol. To build this we will use the socket API. A socket is an interface allowing interprocess communication. It opens up a file descriptor and enables operations similar to as if you were opening up a file (like `read()`, `write()`, and `close()`). In a web server, this enables communication between a process on a client and a process on a server. In the client-server model, a server must first open up a socket that it is listening to on its machine. For a client to talk to the server machine, it must know the IP address of the machine as well as the port that the IP address is exposed to. Below is an example of a web server in Go:

<br>
<script src="https://gist.github.com/kunaljaydesai/bcfe841f552c3074206150b504f5d046.js"></script>
<br>

Under the hood, `net.Listen` makes a couple system calls. The first thing it does is create a socket using the `socket()` syscall. Once the socket has been created, a new file descriptor for that socket is created for that process. Since when a socket will receive data is unpredictable, operations on the file descriptor for the socket are set to non-blocking. This means that if there is no data available in the socket, then the operation won’t block and will continue to process. This would allow our server to do other work (if it needed to, till a connection arrives). Next, the file descriptor is bound to the socket address we specified. In our case, that is `0.0.0.0:8080`. Once the file descriptor is bound to the socket, we call the `listen()` syscall to start accepting connections. Finally, in an infinite-loop, we call the ``accept()`` function which begins to accept connections.

One thing that we should note is that although the file descriptor on the socket is set to non blocking, the `accept()` call in the net package is blocking. To be more clear, the `accept()` syscall is actually non-blocking, however, the net package continues to poll the file descriptor till the `accept()` syscall returns data. This can be seen <a href="https://github.com/golang/go/blob/master/src/internal/poll/fd_unix.go#L367">here</a>.

When the `accept()` call returns with no data, it returns an error called `EAGAIN` or `EWOULDBLOCK` which signals that the call would have blocked if the file descriptor were not on a non-blocking mode. The `net` package in Go simply retries till it receives data.

The `accept()` syscall returns a new file descriptor which points to a new socket specifically for this connection between the client and this server. It is important to note this socket is bound to a different port and the rest of the communication for this connection happens on that socket while the original listening socket is able to still accept incoming connections. When we are handling a specific connection, we call the `read()` syscall. This call blocks until the `FIN` message is received from the client. This way, the server knows it has received all the data from the client, we can load the response into user memory and identify an appropriate response. We write this response to the socket along with a `FIN` once all the data has been sent and both parties close the connection. Once the data has been written to the socket, we can close the socket connection.

You will notice that our web server isn’t very scalable. In fact, it can really only handle one request at a time. How do we make our web server more scalable? In the next post, we will talk about how to build a web server that enables higher concurrency (handling multiple requests at once).