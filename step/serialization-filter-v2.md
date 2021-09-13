# Serialization Filters

## Overview

In this 15-minute lab you will explore the issues of serialization in Java and new features that help developers address these issues.  

### Serialization in Java

Serialization is the process of transforming an object graph into a bytestream. Appropriately the process of deserialization is the inverse, where a bytestream is converted into an object graph.

In the '80s and '90s serialization was a difficult problem, requiring developers to write a lot of hard-to-read and often error-prone code. To address this concern, serialization was implemented as a core Java feature and was vital for its rise in popularity; allowing for the easy transmission and receiving of data to and from remote processes, persisting state to disk, and communication with external services like a database. 

#### Issues with Serialization in Java

While serialization was vital to Java's success, it has also been a source of many headaches. The headaches stem from many issues in how serialization was implemented in Java. A shortlist of issues include;

* Breaks encapsulation by not calling constructors and setting fields reflectively.
* The behavior of serialization depends on the "magic" methods and fields; `readObject`, `writeObject`, `readObjectNoData`, `readResolve`, `writeReplace`, `serialVersionUID`, and `serialPersistentFields`
* Poor stream format that is not efficient, reusable, nor human-readable 

In this lab, you will see some shortcomings of Java serialization, but more importantly how serialization filters address some of these concerns, and how Records are coping with serialization. 

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

public class Message implements Serializable {
	private final String message;

	public Message(String message) {
		this.message = message;
	}

	public String message() {
		return message;
	}
}
</copy>
```

To illustrate serialization, you will use a simple Unix-Domain Socket-Channel based client/server setup.

Create a Server class:

```
<copy>
nano SerializationServer.java
</copy>
```

Paste the following code in `SerializationServer.java`:

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

Create a Client...

```
<copy>
nano SerializationClient.java
</copy>
```

...and paste it the following code:

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

To execute the application, first compile the `Message` class:

```
<copy>
javac Message.java
</copy>
```

Next, start the Server as a background process:

```
<copy>
java SerializationServer.java &
</copy>
```

And then run the Client:

```
<copy>
java SerializationClient.java
</copy>
```

The Server should get the `HELLO WORLD!` message sent by the Client.

💡 You can kill the Client using 'Control','c'.

### Injecting Code with Serialization
One of the (serious!) concerns with serialization is the ability to perform code injection through what is called a [gadget chain](https://blog.redteam-pentesting.de/2021/deserialization-gadget-chain/) attack.

To illustrate this, create a new class called `Exploit`:

```
<copy>
nano Exploit.java
</copy>
```

And copy in the following code: 

```java
<copy>
import java.io.IOException;
import java.io.ObjectInputStream;
import java.io.Serializable;

public class Exploit implements Serializable {

	private String message;
	private interface CodeInjector extends Runnable, Serializable {}

	public Exploit(String message) {
		this.message = message;
	}

	public String getMessage() {
		return message;
	}

	private CodeInjector injectedCode = new CodeInjector() {

		@Override
		public void run() {
			System.out.println("Injected code executed! You have been Rick Rolled!");
		}
	};

	private final void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException {
		in.defaultReadObject();
		injectedCode.run();
	}
}
</copy>
```

Compile `Exploit`:

```
<copy>
javac Exploit.java
</copy>
```

`Exploit` defines and implements an instance of `CodeInjector` an interface that extends `Runnable` and `Serializable`. `Exploit` also defines its own `readObject()`, one of the "magic" methods that controls the deserialization of an object. In `readObject()` the `run()` of `CodeInjector` is being called. 

What makes this dangerous is that an object is deserialized before its type is checked. Let's see this in action.  

Update `SerializationClient`:

```
<copy>
nano SerializationClient.java
</copy>
```

To send `Exploit` instead of `Message` by changing this line:

```java
Message message = new Message("Hello World!");
```

to this:

```java
<copy>
Exploit message = new Exploit("Hello World!");
</copy>
```

You can now start the `SerializationServer` again:

```
<copy>
java SerializationServer.java &
</copy>
```

and run the Client:

```
<copy>
java SerializationClient.java
</copy>
```

You will get the below error message, but notice how the message defined in `CodeInjector.run()` is printed before the `ClassCastException`:

```
Injected code executed! You have been Rick Rolled!
Exception in thread "main" java.lang.ClassCastException: class Exploit cannot be cast to class Message (Exploit and Message are in unnamed module of loader 'app')
	at SerializationServer.main(SerializationServer.java:27)
```

Let's see how serialization filters can protect our code from such issues. 

## Serialization Filters

Java has made serialization easy, but as the above demonstrates, also unsafe! As a consequence writing and maintaining defensive code that secures your application from deserialization attacks can be hard and time-consuming.

The introduction of serialization filters with [JEP 290: Filter Incoming Serialization Data](https://openjdk.java.net/jeps/290) in JDK 9 and [JEP 415: Context-Specific Deserialization Filters](https://openjdk.java.net/jeps/415) in JDK 17, greatly simplified the process of securing your Java applications from deserialization attacks.

💡 You will often see the term "Serialization Filter" although those filters are always applied during deserialization, i.e. when the incoming bytestream is deserialized.

### Setting Serialization Filters using JVM Arguments

Serialization filters can be configured through the command line with the `jdk.serialFilter` JVM argument to accept or reject a bytestream based on a type. 

Let's configure a serialization filter to block the `Exploit` type:

```no-highlight
<copy>
java -Djdk.serialFilter=\!Exploit SerializationServer.java &
</copy>
```

Running the same Client again...

```no-highlight
<copy>
java SerializationClient.java
</copy>
```

...will cause the Server to throw this exception:

```no-highlight
Exception in thread "main" java.io.InvalidClassException: filter status: REJECTED
```

Note though how the message in `CodeInjector.run()` is not displayed. This is because serialization filter rejected it before attempting to deserialize the bytestream.

There are other ways to filter incoming byestreams, for example by size. To configure a serialization filter to reject any bytestream longer than 8 bytes in size, run the Server with the following filter:

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

This will cause the Server to throw again the same exception:

```no-highlight
Exception in thread "main" java.io.InvalidClassException: filter status: REJECTED
```
... which confirms that the Server has rejected the incoming bytestream, our code is safe from those attacks!

### Using the Serialization Filter API

Filters can also be defined programmatically with the `ObjectInputFilter` interface API and added to the `ObjectInputStream` with `setObjectInputFilter(ObjectInputFilter)`. `ObjectInputFilter` can be implemented directly, however there is also the factory method `ObjectInputFilter.Config.createFilter(String)` which takes in a pattern. 

To create a filter that rejects the `Exploit` type, you would do the following: 

Update `SerializationServer.java`:

```
<copy>
nano SerializationServer.java
</copy>
```

When creating the `in` ObjectInputStream, define the filter on it using the following code:

```java
<copy>
ObjectInputStream in = new ObjectInputStream(byteInput);
in.setObjectInputFilter(ObjectInputFilter.Config.createFilter("!Exploit"));
</copy>
```

Start the Server again.

```no-highlight
<copy>
java SerializationServer.java &
</copy>
```

Running the client again...

```no-highlight
<copy>
java SerializationClient.java
</copy>
```

... should cause the Server to throw an exception informing us that incoming bytestream data has been rejected, and hence the Server has not deserialized it! Again, our code is protected from that attack.

```no-highlight
Exception in thread "main" java.io.InvalidClassException: filter status: REJECTED
```

As you saw, Serialization Filters are easy to set up and should be used with any code that is using deserialization. For more information on Serialization Filtering, see here: [https://docs.oracle.com/en/java/javase/16/core/serialization-filtering1.html](https://docs.oracle.com/en/java/javase/16/core/serialization-filtering1.html)

## How Records Address Serialization

When developers need to serialize and deserialize an object graph, in the vast majority of cases the goal is the transmission of the *data* contained within the object graph, not the programmatic behavior in the object graph. As you see the desire to try to reconstitute the entire object graph is the source of many of the woes with Java serialization.

Introduced in Java 16, [JEP 395](https://openjdk.java.net/jeps/395), Records address the outstanding issues with serialization by being transparent carriers of data. To achieve that goal, Records have several design constraints including:

* A Record's superclass is always `java.lang.Record`
* Records are implicitly `final` and cannot be `abstract`
* All fields in a record are `final` and derived from the components defined in its declaration
* Instance fields cannot be declared and cannot have instance initializers 
* Explicit declarations of a derived member must match the type of automatically derived member exactly
* Cannot contain `native` methods

These constraints allow Records to be serialized from its public accessors and deserialized using its canonical constructor. This also means that the serialization and deserialization of a Record class cannot be modified by implementing any of the "magic" methods: `writeObject`, `readObject`, `readObjectNoData`, `writeExternal`, or `readExternal`.

These constraints and behaviors of Records close the loop on many of the encapsulation concerns of serialization and deserialization. 

## Wrap-up

In this exercise, you learned about some of the issues with serialization in Java and how to use some recently introduced features in Java to address these concerns.

For more information on serialization in Java, please check the following resources: 

* [JEP 290: Filter Incoming Serialization Data](https://openjdk.java.net/jeps/290)

* [JEP 415: Context-Specific Deserialization Filters](https://openjdk.java.net/jeps/415)

* [JEP 395: Records](https://openjdk.java.net/jeps/395)

* [Towards Better Serialization](https://cr.openjdk.java.net/~briangoetz/amber/serialization.html) by Brian Goetz

* [Record Serialization in Practice](https://inside.java/2021/04/06/record-serialization-in-practise/) by Julia Boes and Chris Hegarty 

* [Why We Hate Java Serialization](https://inside.java/2019/11/07/whywehateserialization/) by Brian Goetz, Stuart Marks

* [Serialization Filtering](https://docs.oracle.com/en/java/javase/16/core/serialization-filtering1.html)

 