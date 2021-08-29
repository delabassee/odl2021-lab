# Serialization

## Overview

Serialization has long been a sticky subject in Java. Brian Goetz describing it as a paradox, a key to Java's success allowing for transparent remoting, but also the implementation of serialization having a number of critical issues. 

A short list of issues include;

* Breaks encapsulation by ignoring constructors and supplying values to fields via reflection 
* The behavior of serialization depends on "magic" methods (including `readObject()` which we will look at later)
* Poor stream format that is not efficent, reusable, nor human readable 

In this lab we will explore in-practice some of the short comings with serialization in Java, how some of these issues have been addressed in recent JEPs, and how Records addresses many of the remaining issues with serialization. 

## What is Serialization and Deserialization?

Serialization is the process of transforming an object into a bytestream. Appropriately the process of deserialization is the inverse, where a bytestream is converted into a Java object.

As mentioned in the intro, the implemention of serialization as a core Java feature was vital to its raise in popularity. It allowed for the easy transmission and receiving of data to a remote processes, persisting state to disk, and communication with external services like a database. Historically this required writing a lot of complex code that was difficult to write and read, and was often error-prone. 

## Serializing a Class

To make a class serializable in Java, you just need to have it implement the `Serializable` interface, which works as a marker interface, and doesn't require implement any methods. Let's create a simple class for holding a message called `Message`:


```
<copy>
nano Message.java
</copy>
```

`Message` will have checking to make sure the field `message` is neither null, nor empty. To that end the default constructor will be private, and for extra assurance it's not used, an exception will be thrown if it's called. In the public constructor, `Message(String message)`, `message` will be checked that it is neither null nor empty. 

```java
<copy>
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

	public String getMessage() {
		return message;
	}

	@Override
	public String toString() {
		return "Message [message=" + message + "]";
	}
}
</copy>
```

To setup the serialization demonstration, we will use a Unix-Domain Socket-Channel.

Create a "Server" class:

```
<copy>
nano SerializationServer.java
</copy>
```

In `SerializationServer.java` copy and past the following: 

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
		var address = UnixDomainSocketAddress.of("mnt/server");
		try (var serverChannel = ServerSocketChannel.open(UNIX)) {
			serverChannel.bind(address);
			try (var clientChannel = serverChannel.accept()) {

				ByteBuffer buffer = ByteBuffer.allocate(1024);
				clientChannel.read(buffer);

				ByteArrayInputStream byteInput = new ByteArrayInputStream(buffer.flip().array());
				ObjectInputStream in = new ObjectInputStream(byteInput);
				in.setObjectInputFilter(new SerializationFilter());
				Message message = (Message)in.readObject();

System.out.println(message.toString().toUpperCase());
			}
		} finally {
			Files.deleteIfExists(address.getPath());
		}
	}
}
</copy>
```

Create a new "Client" class:

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
		var address = UnixDomainSocketAddress.of("mnt/server");
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

Next start the "Server":

```
<copy>
java SerializationServer.java
</copy>
```

And then run the "Client":

```
<copy>
java SerializationClient.java
</copy>
```

In the console you should get back `HELLO WORLD!`.

### Breaking Encapsulation
Right now everything is behaving as explected, let's see how serialization breaks encapsulation. 

Start the "Server" again:

```
<copy>
java SerializationClient.java
</copy>
```

However before running the client, complete the following steps.

Update `Message`:

```
<copy>
nano Message.java
</copy>
```

By commenting the following, code:

```java
<copy>
import java.io.IOException;
import java.io.ObjectInputStream;
import java.io.Serializable;

public class Message implements Serializable {

	private final String message;
//	private Message() throws OperationNotSupportedException {
//		throw new OperationNotSupportedException();
//	}
	public Message(String message) {
//		if (message == null || message.isEmpty()) {
//			throw new IllegalArgumentException("Message cannot be null or empty!");
//		}
		this.message = message;
	}

	public String getMessage() {
		return message;
	}

	@Override
	public String toString() {
		return "Message [message=" + message + "]";
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

Update `SerializationClient` to set `Message` to a null value:

```
<copy>
nano SerializationClient.java
</copy>
```

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
		var address = UnixDomainSocketAddress.of("mnt/server");
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

This will cause `SerializationServer` to throw a `NullPointerException` and a very confused developer as it should be possible for the field `message` to be null! 

```no-highlight
Exception in thread "main" java.lang.NullPointerException: Cannot invoke "String.toUpperCase()" because the return value of "Object.toString()" is null
	at SerializationServer.main(SerializationServer.java:32)
``` 

This happens because in Java when a stream is being deserialized, the constructor for the class is not called and the fields are set reflectively. So even though in the version of `Message` being used by `SerializationServer` it should programmatically be impossible for `message` to be null. Serialization is able to break encapsulation and set the `message` field to null, creating a difficult to track down bug. 

### Injecting Code with Serialization
The encapsulation breaking behavior of serialization is problemmatic, and the above example is but one simple example of how to break it. However the real security risk primarily comes from the ability to inject code via serialization. Let's take a look at an example of a [gadget chain](https://blog.redteam-pentesting.de/2021/deserialization-gadget-chain/) attack.

Create a new class `Exploit`:

```
<copy>
nano Exploit.java
</copy>
```

And copy in the following:

```java
import java.io.IOException;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.io.Serializable;

public class Exploit implements Serializable {

	private final String message;
	private interface CodeInjector extends Runnable, Serializable {}
	public Exploit(String message) {
		if (message == null || message.isEmpty()) {
			throw new IllegalArgumentException("Message cannot be null or empty!");
		}
		this.message = message;
	}

	public String getMessage() {
		return message;
	}

	public void setMessage(String message) {
		if (message == null || message.isEmpty()) {
			throw new IllegalArgumentException("Message cannot be null or empty!");
		}
		this.message = message;
	}

	private CodeInjector injectedCode = new CodeInjector() {

		@Override
		public void run() {
			System.out.println("Injected code executed, you have been hacked!");
		}
	};

	@Override
	public String toString() {
		return "Message [message=" + message + "]";
	}

	private final void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException {
		in.defaultReadObject();
		injectedCode.run();
	}
}
```

Compile `Exploit`:

```
<copy>
javac Exploit.java
</copy>
```

`Exploit` defines and implements an instance of `CodeInjector` an interface that extends `Runnable` and `Serializable`. `Exploit` also defines its own `readObject()` method, which is one of the methods that controls serialization behavior. Most of the time developers just use the default one create by the java compiler. However a custom `readObject()` can also be defined in a `Serializable` class that will be called instead. In this example `readObject()` calls the runnable: `injectedCode.run();` in its body.

Update `SerializationClient` to send `Exploit` instead:

```
<copy>
nano SerializationClient.java
</copy>
```
Copy/Paste the following:

```
<copy>
import java.io.ByteArrayOutputStream;
import java.io.ObjectOutputStream;
import java.io.Serializable;
import java.net.UnixDomainSocketAddress;
import java.nio.ByteBuffer;
import java.nio.channels.SocketChannel;

public class SerializationClient {

	public static void main(String[] args) throws Exception {
		var address = UnixDomainSocketAddress.of("mnt/server");
		try (var clientChannel = SocketChannel.open(address)) {

			ByteArrayOutputStream out = new ByteArrayOutputStream();
			Exploit message = new Exploit("Hellow World!");
			ObjectOutputStream objectStream = new ObjectOutputStream(out);
			objectStream.writeObject(message);
			ByteBuffer buf = ByteBuffer.wrap(out.toByteArray());

			clientChannel.write(buf);
		}
	}
}
</copy>
```

Start the "Server":

```
<copy>
java SerializationServer.java
</copy>
```

Run the "client":

```
<copy>
java SerializationClient
</copy>
```

In the console the following should be printed:

```
Injected code executed, you have been hacked!
Exception in thread "main" java.lang.ClassCastException: class Exploit cannot be cast to class Message (Exploit and Message are in unnamed module of loader 'app')
	at SerializationServer.main(SerializationServer.java:27)
```

Even though `SerializationServer` is expecting `Message`, `Exploit.readObject()` is still executed and the "exploit" code is run. There is however a way to protect against this expolit. 

### JEP 290

[JEP 290](https://openjdk.java.net/jeps/290) was added in JDK 9, and allows for options of definitions of filters on incoming serialization data to improve security and robustness. 

In this exercise we will define a filter that prevents deserialization of `Runnable`. 

Update `SerializationServer`:

```
<copy>
nano SerializationServer.java
</copy>
```

With the following:

```java
<copy>
public class SerializationServer {
	public static void main(String[] args) throws Exception {
		var address = UnixDomainSocketAddress.of("mnt/server");
		try (var serverChannel = ServerSocketChannel.open(UNIX)) {
			serverChannel.bind(address);
			try (var clientChannel = serverChannel.accept()) {

				ByteBuffer buffer = ByteBuffer.allocate(1024);
				clientChannel.read(buffer);

				ByteArrayInputStream byteInput = new ByteArrayInputStream(buffer.flip().array());
				ObjectInputStream in = new ObjectInputStream(byteInput);
				in.setObjectInputFilter(new SerializationFilter());
				Message message = (Message) in.readObject();

				System.out.println(message.toString().toUpperCase());
			}
		} finally {
			Files.deleteIfExists(address.getPath());
		}
	}
	
	static class SerializationFilter implements ObjectInputFilter {

		public Status checkInput(FilterInfo filterInfo) {
			for (Class typeInterface : filterInfo.serialClass().getInterfaces()) {
				if (typeInterface == Runnable.class) {
					return Status.REJECTED;
				}
			}
			for (Class classField : filterInfo.serialClass().getDeclaredClasses()) {
				for (Class typeInterface : classField.getInterfaces()) {
					if (typeInterface == Runnable.class) {
						return Status.REJECTED;
					}
				}
			}
			return Status.ALLOWED;
		}
	}
}
</copy>
```

In `SerializationServer`, a `SerializationFilter` is defined that will work down the tree of any incoming serialized data looking for the `Runnable` type and if found, reject it. 

Start the "Server":

```
<copy>
java SerializationServer.java
</copy>
```

And run the "Client":

```
<copy>
java SerializationClient.java
</copy>
```

Now the filter will reject the incoming serialized data because of the presence of `Runnable`: 

```
Exception in thread "main" java.io.InvalidClassException: filter status: REJECTED
	at java.base/java.io.ObjectInputStream.filterCheck(ObjectInputStream.java:1360)
	at java.base/java.io.ObjectInputStream.readNonProxyDesc(ObjectInputStream.java:2015)
	at java.base/java.io.ObjectInputStream.readClassDesc(ObjectInputStream.java:1872)
	at java.base/java.io.ObjectInputStream.readOrdinaryObject(ObjectInputStream.java:2179)
	at java.base/java.io.ObjectInputStream.readObject0(ObjectInputStream.java:1689)
	at java.base/java.io.ObjectInputStream.readObject(ObjectInputStream.java:495)
	at java.base/java.io.ObjectInputStream.readObject(ObjectInputStream.java:453)
	at SerializationServer.main(SerializationServer.java:27)
```

**Extra Credit**: You can further confirm that the filter will only reject `Runnable` by updating `SerializationClient` to send `Message` and see that the code executes successfully. 

## How Records Fixes Serialization

Records introduced in Java 16, do a lot to address the many issues demonstrated above with serialization in Java, as well as several other issues not covered in this lab.

* Record components govern seralization (can't have hidden malicious fields)
* Canonical constructor governs deserialization
* "magic" serialization methods; readObject, writeObject, etc. not called

### Filtering for only Records

* Configure serialization filters to only allow classes that extend java.lang.Record

**More resources**

* [JEP 395: Records](https://openjdk.java.net/jeps/395)

* [JEP 290: Filter Incoming Serialization Data](https://openjdk.java.net/jeps/290)

* [JEP 415: Context-Specific Deserialization Filters](https://openjdk.java.net/jeps/415)

* [Towards Better Serialization](https://cr.openjdk.java.net/~briangoetz/amber/serialization.html) by Brian Goetz

* [Record Serialization in Practice](https://inside.java/2021/04/06/record-serialization-in-practise/) by Julia Boes and Chris Hegarty 

* [Why We Hate Java Serialization](https://inside.java/2019/11/07/whywehateserialization/) by Brian Goetz, Stuart Marks (VIDEO)

 