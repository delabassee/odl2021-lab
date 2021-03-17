# Lab 13: Unix Domain Sockets

## Overview

In this 10-minutes lab, you will use **Unix domain sockets**, an addition to the existing [SocketChannel](https://docs.oracle.com/en/java/javase/16/docs/api/java.base/java/nio/channels/SocketChannel.html)/[ServerSocketChannel](https://docs.oracle.com/en/java/javase/16/docs/api/java.base/java/nio/channels/ServerSocketChannel.html) API.

This API provides blocking and multiplexed non-blocking access to sockets. Before Java 16, this was limited to TCP/IP sockets - now it is also possible to access [Unix domain sockets](https://en.wikipedia.org/wiki/Unix_domain_socket). Contrary to traditional sockets, Unix domain sockets are addressed by filesystem path names and are used for inter-process communication on the same host.

💡 Unix domain sockets are supported in Unix-based operating systems (Linux, macOS) and - despite their name - on Windows 10 and Windows Server 2019 too!

## Unix Domain Socket in a Nutshell

As mentioned, Unix domain sockets are based on path names, so the first thing we need is a path that is then turned into a Unix domain socket address. 

```java
// this can be any path!
// here, we're using the home directory to
// make sure we have the right permissions
Path socketFile = Path.of(System.getProperty("user.home")).resolve("server.socket");
UnixDomainSocketAddress address = UnixDomainSocketAddress.of(socketFile);
```
On the server-side, the first step is to create a UDS server socket channel, bind that server socket channel to the UDS address, and then tell it to start accepting a connection.

```java
// server-side
ServerSocketChannel server = ServerSocketChannel.open(StandardProtocolFamily.UNIX);
serverChannel.bind(address);
SocketChannel channel = serverChannel.accept();
```

On the client-side, we need to create a UDS socket channel that we can then use to connect to the UDS address. 

```java
// client-side
SocketChannel channel = SocketChannel.open(StandardProtocolFamily.UNIX);
channel.connect(address);
```

Once the connection is established, the client and the server can send and receive payloads.


## A simple UDS Client & Server

Create a directory.

```nohighlight
<copy>mkdir ~/uds && cd ~/uds</copy>
```

Create the server

```nohighlight
<copy>nano Server.java</copy>
```

with the following content.

```java
<copy>
import java.io.*;
import java.net.*;
import java.nio.*;
import java.nio.channels.*;
import java.nio.file.*;
import java.util.*;

public class Server {

	public static void main(String[] args) throws IOException, InterruptedException {

		Path socketFile = Path.of(System.getProperty("user.home")).resolve("server.socket");
		// in case the file is left over from the last run,
		Files.deleteIfExists(socketFile);
		
		UnixDomainSocketAddress address = UnixDomainSocketAddress.of(socketFile);
		ServerSocketChannel serverChannel = ServerSocketChannel.open(StandardProtocolFamily.UNIX);
		serverChannel.bind(address);
		
		System.out.println("[INFO] Waiting for client to connect...");
		SocketChannel channel = serverChannel.accept();
		System.out.println("[INFO] Client connected");
		
	}
}
</copy>
```

Similarly, create the client

```nohighlight
<copy>nano Client.java</copy>
```

with the following content.


```	
<copy>
import java.io.*;
import java.net.*;
import java.nio.*;
import java.nio.channels.*;
import java.nio.file.*;

public class Client {

	public static void main(String[] args) throws IOException, InterruptedException {

		Path file = Path.of(System.getProperty("user.home")).resolve("server.socket");
		UnixDomainSocketAddress address = UnixDomainSocketAddress.of(file);
		
		SocketChannel channel = SocketChannel.open(StandardProtocolFamily.UNIX);
		channel.connect(address);
		
		// start sending or receiving payload
	}
}

</copy>
```

To test this, you need two terminals, i.e. one for each peer. In the first termainl, launch the server as follows.

```
$ java Server.java
> [INFO] Waiting for client to connect...
```

As you can see, the server launches and then waits for an incomming connection.
Now launch the client in the second terminal.

```nohighlight
<copy>
java Client.java
</copy>
```

Because the server is already running, the client immediately connects to it. The client then exits.
Checking back with the first terminal, you will see that the server registered the connection and shut down too - that's because neither the client nor the server passes any payload.
Let's fix that.

## Exchanging Payload


The [`SocketChannel`](https://docs.oracle.com/en/java/javase/16/docs/api/java.base/java/nio/channels/SocketChannel.html) and [`ServerSocketChannel`](https://docs.oracle.com/en/java/javase/16/docs/api/java.base/java/nio/channels/ServerSocketChannel.html) classes have existed since Java 1.4.  The type of sockets used makes no difference in how payloads are exchanged between them. That means the following code is not specific to Unix domain sockets and works the same with TCP/IP.

Once the client has established the connection, both the server and the client can send and receive payload, i.e. the channel is bi-directional. For simplicity's sake, we will simply send some payload from the client to the server.

### Sending Payload

For the client to send payload, we need to create a [`ByteBuffer`](https://docs.oracle.com/en/java/javase/16/docs/api/java.base/java/nio/ByteBuffer.html), fill it with the payload's bytes, flip it for sending, and then write it to the channel.

Add the following `writeMessageToSocket` method to the Client.java class.


```java
<copy>
private static void writeMessageToSocket(SocketChannel socketChannel, String message) throws IOException {

	ByteBuffer buffer= ByteBuffer.allocate(1024);
	buffer.clear();
	buffer.put(message.getBytes());

	buffer.flip();

	while (buffer.hasRemaining()) {
		socketChannel.write(buffer);
	}
}
</copy>
```
The client can now use this method to send a few messages to the server, ex. by updating its `main` method as follow.


```java
public static void main(String[] args) throws IOException, InterruptedException {
	// ...
    <copy>
	Thread.sleep(3_000);
	writeMessageToSocket(channel, "Hello");

	Thread.sleep(1_000);
	writeMessageToSocket(channel, "Unix domain sockets");
</copy>
}

```

### Receiving Payload

On the receiving side, we do similar steps, but in reverse: read from the channel, flip the bytes, turn them into a `String` message.
Add this `readMessageFromSocket` method to the Server.java class.

```java
<copy>
private static Optional<String> readMessageFromSocket(SocketChannel channel) throws IOException {
	ByteBuffer buffer = ByteBuffer.allocate(1024);
	int bytesRead = channel.read(buffer);
	if (bytesRead < 0)
		return Optional.empty();
	byte[] bytes = new byte[bytesRead];
	buffer.flip();
	buffer.get(bytes);
	String message = new String(bytes);
	return Optional.of(message);
}
</copy>
```
and update the `main` method as follow.

```java

public static void main(String[] args) throws IOException, InterruptedException {
	// ...
    <copy>
	while (true) {
		readMessageFromSocket(channel).ifPresent(System.out::println);
		Thread.sleep(100);
	}
	</copy>	
}
```
This snippet creates an infinite loop that checks every 100 ms whether a new message was written to the socket and, if so, outputs it.


To test this, launch the server in one terminal (`java Server.java`), and the client in another termainal (`java Client.java`). The server should receive the message sent by the client.

💡 Shutdown the server with [CTRL]+[C].


## Real-life Complexities
The code presented in this exercise barely scratches the surface of how to implement communication via sockets. 
For the sake of simplicity, the code has been over-simplified. You will notice that the server always needs to be launched first, that it can only accept one connection, that as soon as the connection is dropped by the client, it can never create a new one (ex. try to launch `Client.java` several times), and that it runs indefinitely until forced to shut down. Moreover, it doesn't do any proper clean-up (like deleting the created `server.socket` file) and neither does the client (by closing the connection). And finally, there's error handling, something that is clearly necessary... even more with inter-processes communication!

Compared to TCP/IP loopback connections, Unix domain sockets offer several advantages:

* Because they can only be used for communication on the same host, opening them instead of a TCP/IP socket has no risk to accept remote connections.
* Access control is applied with file-based mechanisms, which are detailed, well understood, and enforced by the operating system.
* Unix domain sockets have faster setup times and higher data throughput than TCP/IP loopback connections.

💡 Unix domain sockets can be used to communicate between containers as long as the UDS socket is created on a shared volume.

## Wrap-up

In this exercise, you have used the SocketChannel/ServerSocketChannel API to establish inter-process communication on the same host with Unix domain sockets, which were added to the pre-existing API in Java 16.
Unix-domain sockets are both more secure and more efficient than TCP/IP loopback connections and are supported on all Unix-based operating systems as well on some modern Windows versions.


**More ressources**

* [Unix domain socket channels overview](https://inside.java/2021/02/03/jep380-unix-domain-sockets-channels/)
* [JEP 380: Unix Domain Socket Channels](https://openjdk.java.net/jeps/380)


