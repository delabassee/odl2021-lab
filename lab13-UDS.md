# Lab 13: Unix Domain Sockets

## Overview
​
In this 15-minutes lab, you will use **Unix domain sockets**, an addition to the existing [SocketChannel](https://download.java.net/java/early_access/jdk16/docs/api/java.base/java/nio/channels/SocketChannel.html)/[ServerSocketChannel](https://download.java.net/java/early_access/jdk16/docs/api/java.base/java/nio/channels/ServerSocketChannel.html) API.
​

This API provides blocking and multiplexed non-blocking access to sockets. Before Java 16, this was limited to TCP/IP sockets - now it is also possible to access Unix domain sockets. They are addressed by filesystem path names, and are used for inter-process communication on the same host.
​

💡 Unix domain sockets are supported in Unix based operating system (Linux, macOS) and - despite their name - on Windows 10 and Windows Server 2019.
​
## Accessing a Unix Domain Socket
​
As mentioned, Unix domain sockets are based on path names, so the first thing we need is a path that we can then turn into a socket address:
​
```java
// this can be any path - here, we're using the home directory
// to make sure we have the required permissions
Path socketFile = Path.of(System.getProperty("user.home")).resolve("server.socket");
UnixDomainSocketAddress address = UnixDomainSocketAddress.of(socketFile);
```
​
The next step is to launch a server and a client on that address:
​
```java
// server
ServerSocketChannel serverChannel = ServerSocketChannel.open(StandardProtocolFamily.UNIX);
serverChannel.bind(address);
​
// client
SocketChannel channel = SocketChannel.open(StandardProtocolFamily.UNIX);
```
​
The third and final step before we can send messages is the client and server connecting to one another:
​
```java
// server
SocketChannel channel = serverChannel.accept();
​
// client
channel.connect(address);
```
​
To make this executable, the server and client need to have their own source files `Server.java` and `Client.java`:
​
```java
// Server.java
import java.io.*;
import java.net.*;
import java.nio.*;
import java.nio.channels.*;
import java.nio.file.*;
import java.util.*;
​
public class Server {
​
	public static void main(String[] args) throws IOException, InterruptedException {
		Path socketFile = Path.of(System.getProperty("user.home")).resolve("server.socket");
		// in case the file is left over from the last run, this makes the demo more robust
		Files.deleteIfExists(socketFile);
		UnixDomainSocketAddress address = UnixDomainSocketAddress.of(socketFile);
​
		ServerSocketChannel serverChannel = ServerSocketChannel.open(StandardProtocolFamily.UNIX);
		serverChannel.bind(address);
​
		System.out.println("[INFO] Waiting for client to connect...");
		SocketChannel channel = serverChannel.accept();
		System.out.println("[INFO] Client connected");
​
		// start receiving messages
	}
​
}
​
// Client.java
import java.io.*;
import java.net.*;
import java.nio.*;
import java.nio.channels.*;
import java.nio.file.*;
​
public class Client {
​
	public static void main(String[] args) throws IOException, InterruptedException {
		Path file = Path.of(System.getProperty("user.home")).resolve("server.socket");
		UnixDomainSocketAddress address = UnixDomainSocketAddress.of(file);
​
		SocketChannel channel = SocketChannel.open(StandardProtocolFamily.UNIX);
		channel.connect(address);
​
		// start receiving messages
	}
​
}
​
```
​
To give this a go, you need two terminals.
In the first one, launch the server as follows:
​
```
$ java Server.java
> [INFO] Waiting for client to connect...
```
​
As you can see, the server launches and then waits for a connection.
Now launch the client in the second terminal:
​
```
java Client.java
```
​
Because the server is already running, the client can immediately connect and exit the program.
Checking back with the first terminal, you will see that the server registered the connection and shut down - that's because neither client nor server pass any messages.
We'll do that next.
​
​
## Passing Messages
​
The `SocketChannel` and `ServerSocketChannel` classes have existed since Java 4 and what kind of socket they use to connect makes no difference in how messages are passed between them.
That means the following code is not specific to Unix domain sockets and works the same with TCP/IP.
​
Both server and client can send and receive messages, but for simplicity's sake, we're just going to send from the client to the server.
​
### Sending Messages
​
For the client to send a message, we need to create a `ByteBuffer`, fill it with the message's bytes, flip it for sending, and then write to the channel:
​
```java
// in Client.java
private static void writeMessageToSocket(SocketChannel socketChannel, String message) throws IOException {
	ByteBuffer buffer= ByteBuffer.allocate(1024);
	buffer.clear();
	buffer.put(message.getBytes());
	buffer.flip();
	while(buffer.hasRemaining()) {
		socketChannel.write(buffer);
	}
}
```
​
We now use this method to send a few messages to the server:
​
```java
// in Client.java
public static void main(String[] args) throws IOException, InterruptedException {
​
	// as above
​
	Thread.sleep(3_000);
	writeMessageToSocket(channel, "Hello");
	Thread.sleep(1_000);
	writeMessageToSocket(channel, "Unix domain sockets");
}
```
​
### Receiving Messages
​
On the receiving side, we do similar steps, but in reverse: read from the channel, flip the bytes, turn them into a message:
​
```java
// in Server.java
private static Optional<String> readMessageFromSocket(SocketChannel channel) throws IOException {
	ByteBuffer buffer = ByteBuffer.allocate(1024);
	int bytesRead = channel.read(buffer);
	if (bytesRead < 0)
		return Optional.empty();
​
	byte[] bytes = new byte[bytesRead];
	buffer.flip();
	buffer.get(bytes);
	String message = new String(bytes);
	return Optional.of(message);
}
```
​
```java
// in Server.java
public static void main(String[] args) throws IOException, InterruptedException {
​
	// as above
​
	while (true) {
		readMessageFromSocket(channel).ifPresent(System.out::println);
		Thread.sleep(100);
	}
}
```
​
This creates an infinite loop that checks every 100 ms whether a new message was written to the socket and, if so, outputs it.
This means that the server will now run indefinitely until you shut it down in the terminal by hitting CTRL-C.
​
If you launch the server and client (in that order) as before, you will now see that the messages sent by the client are printed to the output by the server.
​
​
## Real-life Complexities
​
The code presented above just scratches the surface of how to implement the communication via sockets.
You will notice that the server always needs to be launched first, that it can only accept one connection, that as soon as that's abandoned by the client, it can never create a new one (try to launch `Client.java` several times), and that it runs indefinitely until forced to shut down.
It does no clean-up (like deleting the created `server.socket` file) and neither does the client (by closing the connection).
​

Compared to TCP/IP loopback connections, Unix domain sockets have several advantages:
​
* Because they can only be used for communication on the same host, opening them instead of a TCP/IP socket has no risk to accept remote connections.
* Access control is applied with file-based mechanisms, which are detailed, well understood, and enforced by the operating system.
* Unix domain sockets have faster setup times and higher data throughput than TCP/IP loopback connections.
​

Note that you can even use Unix domain sockets for communication between containers on the same system as long as you create the sockets on a shared volume.
​
​
## Wrap-up
​
In this exercise, you have used the SocketChannel/ServerSocketChannel API to establish inter-process communication on the same host with Unix domain sockets, which were added to the pre-existing API in Java 16.
Unix-domain sockets are both more secure and more efficient than TCP/IP loopback connections and supported on all Unix-based operating system as well on some modern Windows versions.


**More ressources**

* [Unix domain socket channels overview](https://inside.java/2021/02/03/jep380-unix-domain-sockets-channels/)

* [JEP 380: Unix Domain Socket Channels](https://openjdk.java.net/jeps/380)



 







