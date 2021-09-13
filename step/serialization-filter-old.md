# Serialization Filters

## Overview

Serialization is the process of transforming an object graph into a bytestream. Appropriately the process of deserialization is the inverse, where a bytestream is converted into an object graph.

In the '80s and '90s serialization was a difficult problem, requiring developers to write a lot of hard to read and often error-prone code. To address this concern, serialization was implemented as a core Java feature and was vital for its rise in popularity; allowing for the easy transmission and receiving of data to and from remote processes, persisting state to disk, and communication with external services like a database. 

### Issues with Serialization in Java

While serialization was vital to Java's success, it has also been a source of many headaches. The headaches stem from many issues in how serialization was implemented in Java. A shortlist of issues include;

* Breaks encapsulation by not calling constructors and setting fields reflectively.
* The behavior of serialization depends on the "magic" methods and fields; `readObject`, `writeObject`, `readObjectNoData`, `readResolve`, `writeReplace`, `serialVersionUID`, and `serialPersistentFields`
* Poor stream format that is not efficient, reusable, nor human-readable 

In this lab, we will see some short comings of serialization in Java, but more importantly how serialization filters address some of these concerns, and how Records are coping with serialization. 

## Serializing a Class

To make a class serializable in Java, you just need to have it implement the `Serializable` interface, which works as a marker interface, and doesn't require implementing any methods. In a new directory, create a simple class for holding a message called `Message`:

```
<copy>
nano Message.java
</copy>
```

with the following content...

```java
<copy>
import java.io.Serializable;

import javax.naming.OperationNotSupportedException;

public class Message implements Serializable {
	private final String message;
	private Message() throws OperationNotSupportedException {
		throw new OperationNotSupportedException();
	}
	public Message(String message) {
		if (message == null || message.isEmpty()) {
			throw new IllegalArgumentException("Message cannot be null or empty!");
		}
		this.message = message;
	}

	public String message() {
		return message;
	}
}
</copy>
```

As seen in the above `Message` has a single field, `private final String message`. The only way to set the field is through the constructor, which has validation checks to make sure the field is neither null, nor empty. Further, the default constructor is private and will throw an exception if called. So programmatically it should not be possible for the field `message` to be null. 

To setup the serialization demonstration, we will use a Unix-Domain Socket-Channel.

Create a "Server" class:

```
<copy>
nano SerializationServer.java
</copy>
```

In `SerializationServer.java` copy and paste the following: 

```java
<copy>
import static java.net.StandardProtocolFamily.UNIX;

import java.io.ByteArrayInputStream;
import java.io.ObjectInputFilter;
import java.io.ObjectInputStream;
import java.io.Serializable;
import java.net.UnixDomainSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.ServerSocketChannel;
import java.nio.file.Files;

import javax.naming.OperationNotSupportedException;

public class SerializationServer {
	public static void main(String[] args) throws Exception {
		var address = UnixDomainSocketAddress.of("server");
		try (var serverChannel = ServerSocketChannel.open(UNIX)) {
			serverChannel.bind(address);
			try (var clientChannel = serverChannel.accept()) {

				ByteBuffer buffer = ByteBuffer.allocate(1024);
				clientChannel.read(buffer);

				ByteArrayInputStream byteInput = new ByteArrayInputStream(buffer.flip().array());
				ObjectInputStream in = new ObjectInputStream(byteInput);
				Message message = (Message)in.readObject();

				System.out.println(message.message().toUpperCase());
			}
		} finally {
			Files.deleteIfExists(address.getPath());
		}
	}
}
</copy>
```

Create a "Client" class:

```
<copy>
nano SerializationClient.java
</copy>
```

And copy/paste the following:

```java
<copy>
import java.io.ByteArrayOutputStream;
import java.io.ObjectOutputStream;
import java.io.Serializable;
import java.net.UnixDomainSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SocketChannel;

public class SerializationClient {

	public static void main(String[] args) throws Exception {
		var address = UnixDomainSocketAddress.of("server");
		try (var clientChannel = SocketChannel.open(address)) {

			ByteArrayOutputStream out = new ByteArrayOutputStream();
			Message message = new Message("Hello World!");
			ObjectOutputStream objectStream = new ObjectOutputStream(out);
			objectStream.writeObject(message);
			ByteBuffer buf = ByteBuffer.wrap(out.toByteArray());

			clientChannel.write(buf);
		}
	}
}
</copy>
```

To execute the application, first compile the `Message`:

```
<copy>
javac Message.java
</copy>
```

Next start the "Server" as a background process:

```
<copy>
java SerializationServer.java &
</copy>
```

And then run the "Client":

```
<copy>
java SerializationClient.java
</copy>
```

You should get back `HELLO WORLD!`.

💡 You can kill the client using 'Control','c'.

### Breaking Encapsulation
Right now everything is behaving as expected, let's see how serialization breaks encapsulation. 

Start the "Server" again as a background process:

```
<copy>
java SerializationServer.java &
</copy>
```

However, before running the client, complete the following steps.

Update `Message`:

```
<copy>
nano Message.java
</copy>
```

By commenting the validation checks in the `public Message(String)`:

```java
<copy>
import java.io.IOException;
import java.io.ObjectInputStream;
import java.io.Serializable;

public class Message implements Serializable {

	private final String message;
	private Message() throws OperationNotSupportedException {
		throw new OperationNotSupportedException();
	}
	public Message(String message) {
//		if (message == null || message.isEmpty()) {
//			throw new IllegalArgumentException("Message cannot be null or empty!");
//		}
		this.message = message;
	}

	public String getMessage() {
		return message;
	}
}
</copy>
```
Recompile `Message`:

```
<copy>
javac Message.java
</copy>
```

Update `SerializationClient`...

```
<copy>
nano SerializationClient.java
</copy>
```

To maliciously pass `null` into the constructor of `Message`:

```java
<copy>
import java.io.ByteArrayOutputStream;
import java.io.ObjectOutputStream;
import java.io.Serializable;
import java.net.UnixDomainSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SocketChannel;

public class SerializationClient {

	public static void main(String[] args) throws Exception {
		var address = UnixDomainSocketAddress.of("server");
		try (var clientChannel = SocketChannel.open(address)) {

			ByteArrayOutputStream out = new ByteArrayOutputStream();
			Message message = new Message(null);
			ObjectOutputStream objectStream = new ObjectOutputStream(out);
			objectStream.writeObject(message);
			ByteBuffer buf = ByteBuffer.wrap(out.toByteArray());

			clientChannel.write(buf);
		}
	}
}
</copy>
```

Now run `SerializationClient`:

```
<copy>
java SerializationClient.java
</copy>
```

This will cause `SerializationServer` to throw a `NullPointerException` and a very confused developer as it shouldn't be possible for the field `message` to be null! 

```no-highlight
Exception in thread "main" java.lang.NullPointerException: Cannot invoke "String.toUpperCase()" because the return value of "Object.toString()" is null
	at SerializationServer.main(SerializationServer.java:32)
``` 

This happens because in Java when a bytestream is being deserialized, it's not the constructor of the class defined in the bytestream that is called, but instead, an empty object is created and the fields are recursively set through reflection. 

The above demonstrates how serialization undermines the integrity of the object graph, creating subtle and difficult to fix bugs.

### Injecting Code with Serialization
Another serious concern with serialization is the ability to perform code injection through what is called a [gadget chain](https://blog.redteam-pentesting.de/2021/deserialization-gadget-chain/) attack.

An example of this is with the below `Exploit` class: 

```java
public class Exploit implements Serializable {
	private interface CodeInjector extends Runnable, Serializable {}

	private CodeInjector injectedCode = new CodeInjector() {

		@Override
		public void run() {
			System.out.println("Injected code executed, you have been hacked!");
		}
	};

	private final void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException {
		in.defaultReadObject();
		injectedCode.run();
	}
}
```

`Exploit` defines and implements an instance of `CodeInjector` an interface that extends `Runnable` and `Serializable`. `Exploit` also defines its own `readObject()`, one of the "magic" methods that controls the deserialization of an object. In `readObject()` the `run()` of `CodeInjector` is being called. 

What makes this dangerous is that an object is deserialized before its type is checked. If `SerializationClient` was updated to send `Exploit` instead of `Message` the below response would be printed by `SerializationServer`, note how the message defined in `CodeInjector.run()` is printed before the `ClassCastException`:

```
Injected code executed, you have been hacked!
Exception in thread "main" java.lang.ClassCastException: class Exploit cannot be cast to class Message (Exploit and Message are in unnamed module of loader 'app')
	at SerializationServer.main(SerializationServer.java:27)
```

The above are couple examples of how serialization can introduce difficult to track bugs, or potential security risks. 

## Serialization Filters

Java has made serialization easy, but as the above demonstrates, also unsafe! As a consequence writing and maintaining defensive code that secures your application from deserialization attacks can be hard and time-consuming.

The introduction of serialization filters with [JEP 290: Filter Incoming Serialization Data](https://openjdk.java.net/jeps/290) in JDK 9 and [JEP 415: Context-Specific Deserialization Filters](https://openjdk.java.net/jeps/415) in JDK 17, greatly simplified the process of securing your Java applications from deserialization attacks.

### Setting Serialization Filters using JVM Arguments

Serialization filters can be configured through the command line with the `jdk.serialFilter` JVM argument. 

Incoming serialized data can be filtered by its size. To configure the serialization filter to reject any bytestream larger than 8 bytes in size run the following:

```no-highlight
<copy>
java -Djdk.serialFilter=maxbytes=8 SerializationServer.java &
</copy>
```

and run the client again:

```no-highlight
<copy>
java SerializationClient.java
</copy>
```

This will cause an exception to be thrown rejecting the incoming serialized data:

```no-highlight
Exception in thread "main" java.io.InvalidClassException: filter status: REJECTED
```

Serialization filters can also be configured to accept or reject based by type. To reject `Message` the following can be done:

```no-highlight
<copy>
java -Djdk.serialFilter=\!Message SerializationServer.java &
</copy>
```

Which when running the client again:

```no-highlight
<copy>
java SerializationClient.java
</copy>
```

will produce the same exception:

```no-highlight
Exception in thread "main" java.io.InvalidClassException: filter status: REJECTED
```

### Using the Serialization Filter API

Filters can also be defined programmatically with the `ObjectInputFilter` interface API and added to the `ObjectInputStream` with `setObjectInputFilter`. `ObjectInputFilter` can be implemented directly, however there is also the factory method `ObjectInputFilter.Config.createFilter(String)` which takes in a pattern. 

To create a filter that rejects on `Message` type, you would do the following: 

Update `SerializationServer.java`:

```
<copy>
nano SerializationServer.java
</copy>
```

Copy the following serialization filter definition:

```java
<copy>
in.setObjectInputFilter(ObjectInputFilter.Config.createFilter("!Message"));
</copy>
```

and add it *after* this line in `SerializationServer.java`:

```
ObjectInputStream in = new ObjectInputStream(byteInput);
```

Start the "server" again: 

```no-highlight
<copy>
java SerializationServer.java &
</copy>
```

run the client again:

```no-highlight
<copy>
java SerializationClient.java
</copy>
```

And an exception rejecting the serialized data will be thrown:

```no-highlight
Exception in thread "main" java.io.InvalidClassException: filter status: REJECTED
```

For more information on Serialization Filtering, see here: [https://docs.oracle.com/en/java/javase/16/core/serialization-filtering1.html](https://docs.oracle.com/en/java/javase/16/core/serialization-filtering1.html)

## How Records Address Serialization

When developers need to serialize and deserialize an object graph, in the vast majority of cases the goal is the transmission of the *data* contained within the object graph, not the programmatic behavior in the object graph. As we see the desire to try to reconstitute the entire object graph is the source of many of the woes with Java serialization.

Introduced in Java 16, [JEP 395](https://openjdk.java.net/jeps/395), Records address the outstanding issues with serialization by being transparent carriers of data. To achieve that goal, Records have several design constraints including:

* A record's superclass is always `java.lang.Record`
* Records are implicitly `final` and cannot be `abstract`
* All fields in a record are `final` and derived from the components defined in its declaration
* Instance fields cannot be declared and cannot have instance initializers 
* Explicit declarations of a derived member must match the type of automatically derived member exactly
* Cannot contain `native` methods

These constraints allow records to be serialized from its public accessors, and deserialized using its canonical constructor. This also means that the serialization and deserialization of a record class cannot be modified by implementing any of the "magic" methods: `writeObject`, `readObject`, `readObjectNoData`, `writeExternal`, or `readExternal`.

These constraints and behaviors of Records close the loop on many of the encapsulation concerns of serialization and deserialization. 

**More resources**

* [JEP 290: Filter Incoming Serialization Data](https://openjdk.java.net/jeps/290)

* [JEP 415: Context-Specific Deserialization Filters](https://openjdk.java.net/jeps/415)

* [JEP 395: Records](https://openjdk.java.net/jeps/395)

* [Towards Better Serialization](https://cr.openjdk.java.net/~briangoetz/amber/serialization.html) by Brian Goetz

* [Record Serialization in Practice](https://inside.java/2021/04/06/record-serialization-in-practise/) by Julia Boes and Chris Hegarty 

* [Why We Hate Java Serialization](https://inside.java/2019/11/07/whywehateserialization/) by Brian Goetz, Stuart Marks

* [Serialization Filtering](https://docs.oracle.com/en/java/javase/16/core/serialization-filtering1.html)

 