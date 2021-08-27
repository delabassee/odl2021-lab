# Serialization

## Overview

Serialization has long been a sticky subject in Java. Brian Goetz describing it as a paradox, a key to Java's success allowing for transparent remoting, but also the implementation of serialization making a number of key mistakes. 

A short list of mistakes include;
* Bypasses constructors and encapsulation  
* The behavior of serialization depends on "magic" methods (including `readObject()` which we will look at later)
* Poor stream format that is not efficent, reusable, nor human readable 

In this section we will explore some of the short comings with Serialization in Java, how some of these issues have been addressed in recent JEPs, and how Records addresses many of the issues with serialization. 

## Serializing a Class

To make a class Serializable in Java, you just need to have it implement the `Serializable` interface. Which works as a marker interface, and doesn't require implement any methods. Let's create a simple class for holding a message called `MessageClass`:


```
<copy>
nano MessageClass.java
</copy>
```

`MessageClass` will have checkingto make sure the field `message` is neither null, nor empty. To that end the default constructor will be private, and for extra assurance it's not used, and exception will be thrown if it's called. In `MessageClass(String message)`, `message` will be be checked that is neither null nor empty, and if either are true and exception will be thrown. 

```java
<copy>
public class MessageClass implements Serializable {
	private String message;
	private MessageClass() throws OperationNotSupportedException {
		throw new OperationNotSupportedException();
	}
	public MessageClass(String message) {
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
		return "MessageClass [message=" + message + "]";
	}
}
</copy>
```
To setup the serialization demonstration, we will use a Unix-Domain Socket-Channel.

To create a new "Server" class:

```
<copy>
nano ClassSerializationServer.java
</copy>
```

In `ClassSerializationServer.java` copy and past the following: 

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

public class ClassSerializationServer {
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
				MessageClass message = (MessageClass)in.readObject();

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
nano ClassSerializationClient.java
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

public class ClassSerializationClient {

	public static void main(String[] args) throws Exception {
		var address = UnixDomainSocketAddress.of("mnt/server");
		try (var clientChannel = SocketChannel.open(address)) {

			ByteArrayOutputStream out = new ByteArrayOutputStream();
			MessageClass message = new MessageClass("Hello World!");
			ObjectOutputStream objectStream = new ObjectOutputStream(out);
			objectStream.writeObject(message);
			ByteBuffer buf = ByteBuffer.wrap(out.toByteArray());

			clientChannel.write(buf);
		}
	}
}
</copy>
```
To execute the application, first compile the `MessageClass`:

```
<copy>
javac MessageClass.java
</copy>
```

Next start the "Server":

```
<copy>
java ClassSerializationServer.java
</copy>
```

And then run the "Client":

```
<copy>
java ClassSerializationClient.java
</copy>
```

In the console you should get back `HELLO WORLD!`.

### Breaking Encapsulation
Right now everything is behaving as explected, let's see how , let see now how serialization breaks encapsulation. 

Start the "Server" again:

```
<copy>
java ClassSerializationServer.java
</copy>
```

However before starting the client, run the following steps.

Update `MessageClass`

```
<copy>
nano MessageClass.java
</copy>
```

With the following, code:

```java
<copy>
import java.io.IOException;
import java.io.ObjectInputStream;
import java.io.Serializable;

public class MessageClass implements Serializable {

	private String message;
//	private MessageClass() throws OperationNotSupportedException {
//		throw new OperationNotSupportedException();
//	}
	public MessageClass(String message) {
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
		return "MessageClass [message=" + message + "]";
	}
}
</copy>
```
Recompile `MessageClass`:

```
<copy>
javac MessageClass.java
</copy>
```

Update `ClassSerializationClient` to set `MessageClass` to a null value:

```
<copy>
nano ClassSerializationClient.java
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

public class ClassSerializationClient {

	public static void main(String[] args) throws Exception {
		var address = UnixDomainSocketAddress.of("mnt/server");
		try (var clientChannel = SocketChannel.open(address)) {

			ByteArrayOutputStream out = new ByteArrayOutputStream();
			MessageClass message = new MessageClass(null);
			ObjectOutputStream objectStream = new ObjectOutputStream(out);
			objectStream.writeObject(message);
			ByteBuffer buf = ByteBuffer.wrap(out.toByteArray());

			clientChannel.write(buf);
		}
	}
}
</copy>
```

Now run `ClassSerializationClient`:

```
<copy>
java ClassSerializationClient.java
</copy>
```

Which will throw a NullPointerException, when in `ClassSerializationServer` `toUpperCase()` is called: 

```no-highlight
Exception in thread "main" java.lang.NullPointerException: Cannot invoke "String.toUpperCase()" because the return value of "Object.toString()" is null
	at ClassSerializationServer.main(ClassSerializationServer.java:32)
``` 

This happens because in Java when a stream is being deserialized, the constructor for the class is not called and the fields are set reflectively. So even though in the version of `MessageClass` being used by `ClassSerializationServer` it should programmatically be impossible for `message` to be null. Serialization is able to break encapsulation and provide a null value, creating a difficult to track down bug. 

### Injecting Code with Serialization
The encapsulation breaking behavior of serialization can be problemmatic, but the real security risk can come from the ability to inject code via serialization. Let's take a look at an example of a [gadget chain](https://blog.redteam-pentesting.de/2021/deserialization-gadget-chain/) attack.

Create a new class `ExploitClass`:

```
<copy>
nano ExploitClass.java
</copy>
```

And copy in the following:

```java
import java.io.IOException;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.io.Serializable;

public class ExploitClass implements Serializable {

	private String message;
	private interface CodeInjector extends Runnable, Serializable {}
	public ExploitClass(String message) {
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
		return "MessageClass [message=" + message + "]";
	}

	private final void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException {
		in.defaultReadObject();
		injectedCode.run();
	}
}
```

Compile `ExploitClass`:

```
<copy>
javac ExplotClass.java
</copy>
```

`ExploitClass` defines and implements an instance of `CodeInjector` an interface that extends `Runnable` and `Serializable`. `ExploitClass` also defines its own `readObject()` method, which is one of the methods that controls serialization behavior. Most of the time developers just use the default one create by the java compiler. However a custom `readObject()` can also be used that will be called instead. In this example `readObject()` calls the runnable: `injectedCode.run();` in its body.

Update `ClassSerializationClient` to send `ExploitClass` instead:

```
<copy>
nano ClassSerializationClient.java
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

public class ClassSerializationClient {

	public static void main(String[] args) throws Exception {
		var address = UnixDomainSocketAddress.of("mnt/server");
		try (var clientChannel = SocketChannel.open(address)) {

			ByteArrayOutputStream out = new ByteArrayOutputStream();
			ExploitClass message = new ExploitClass("Hellow World!");
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
java ClassSerializationServer.java
</copy>
```

Run the "client":

```
<copy>
java ClassSerializationClient.java
</copy>
```

In the console the following should be printed:

```
Injected code executed, you have been hacked!
Exception in thread "main" java.lang.ClassCastException: class ExploitClass cannot be cast to class MessageClass (ExploitClass and MessageClass are in unnamed module of loader 'app')
	at ClassSerializationServer.main(ClassSerializationServer.java:27)
```

Even though `ClassSerializationServer` is expecting `MessageClass`, `ExploitClass.readObject()` is still executed and the "exploit" code is run. There is however a way to protect agains this expolit. 

### JEP 290

[JEP 290](https://openjdk.java.net/jeps/290) was added in JDK 9, and allows for options of definitions of filters on incoming serialization data to improve security and robustness. 

In this exercise we will define a filter that prevents deserialization of `Runnable`. 

Update `ClassSerializationServer`:

```
<copy>
nano ClassSerializationServer.java
</copy>
```

With the following:

```java
<copy>
public class ClassSerializationServer {
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
				MessageClass message = (MessageClass) in.readObject();

				System.out.println(message.toString().toUpperCase());
			}
		} finally {
			Files.deleteIfExists(address.getPath());
		}
	}

	
	static class MessageClass implements Serializable {

		private static final long serialVersionUID = 1L;
		private String message;

		private MessageClass() throws OperationNotSupportedException {
			throw new OperationNotSupportedException();
		}
		
		public MessageClass(String message) {
			if (message == null || message.isEmpty()) {
				throw new IllegalArgumentException("Message cannot be null or empty!");
			}
			this.message = message;
		}
		
		@Override
		public String toString() {
			return message;
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
In `ClassSerializationServer` `SerializationFilter` is defined that will work down the tree of any incoming serialized data for the `Runnable` type and if found, reject it. 

Start the "Server":

```
<copy>
java ClassSerializationServer.java
</copy>
```
And run the "Client":

```
<copy>
java ClassSerializationClient.java
</copy>
```

Now the filter will reject the incoming serialized data becuase of the presence of `Runnable`: 

```
Exception in thread "main" java.io.InvalidClassException: filter status: REJECTED
	at java.base/java.io.ObjectInputStream.filterCheck(ObjectInputStream.java:1360)
	at java.base/java.io.ObjectInputStream.readNonProxyDesc(ObjectInputStream.java:2015)
	at java.base/java.io.ObjectInputStream.readClassDesc(ObjectInputStream.java:1872)
	at java.base/java.io.ObjectInputStream.readOrdinaryObject(ObjectInputStream.java:2179)
	at java.base/java.io.ObjectInputStream.readObject0(ObjectInputStream.java:1689)
	at java.base/java.io.ObjectInputStream.readObject(ObjectInputStream.java:495)
	at java.base/java.io.ObjectInputStream.readObject(ObjectInputStream.java:453)
	at ClassSerializationServer.main(ClassSerializationServer.java:27)
```

**Extra Credit**: You can further confirm that the filter will only reject `Runnable` by updating `ClassSerializationClient` to send `MessageClass` and see that the code executes successfully. 

## How Records Fixes Serialization

## Serializing a Record

Records are 

```java
public record MessageRecord(String message) {
}
```

**More resources**

* [JEP 395: Records](https://openjdk.java.net/jeps/395)

* [JEP 290: Filter Incoming Serialization Data](https://openjdk.java.net/jeps/290)

* [Towards Better Serialization](https://cr.openjdk.java.net/~briangoetz/amber/serialization.html) by Brian Goetz

* [Serialization and deserialization in Java: explaining the Java deserialize vulnerability](https://snyk.io/blog/serialization-and-deserialization-in-java/) by Brian Vermeer


 