# Serialization

## Overview

Serialization has long been a sticky subject in Java. Brian Goetz describing it as a paradox, a key to Java's success allowing for transparent remoting, but also the implementation of serialization making a number of key mistakes. 

A short list of mistakes with serialization include;
* Bypasses constructors and accessibility  
* The behavior of serialization depends on "magic" methods (including `readObject()` and `writeObject()`, which we will look at later, among others)
* Poor stream format that is not efficent, reusable, or human readable 

In this section we will explore some of the short comings with Serialization as well as how Records address these issues. 

## Serializing a Class
```java
public class MessageClass implements Serializable {

	private static final long serialVersionUID = 1677690843846149591L;
	private String message;

	public MessageClass(String message) {
		this.message = message;
	}

	public String getMessage() {
		return message;
	}

	public void setMessage(String message) {
		this.message = message;
	}
}
```


## Serializing a Record

```java
public record MessageRecord(String message) {
}
```

## Finding the Cracks in Serialization

### Bypassing Encapsulation
```java
import java.io.IOException;
import java.io.ObjectInputStream;
import java.io.Serializable;

public class MessageClass implements Serializable {

	private static final long serialVersionUID = 7220834928776118380L;
	private String message;


	public MessageClass(String message) {
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



	@Override
	public String toString() {
		return "MessageClass [message=" + message + "]";
	}
	private final void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException {		
		message = null;
	}
}
```


### Injecting Code

```java
import java.io.IOException;
import java.io.ObjectInputStream;
import java.io.Serializable;

public class MessageClass implements Serializable {
	private interface NefariousCode extends Runnable, Serializable {};

	private static final long serialVersionUID = 7220834928776118380L;
	private String message;


	public MessageClass(String message) {
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



	@Override
	public String toString() {
		return "MessageClass [message=" + message + "]";
	}

	private Runnable command = new NefariousCode() {
		@Override
		public void run() {
			System.out.println("Injected Code");
		}
	};
	private final void readObject(ObjectInputStream in) throws IOException, ClassNotFoundException {		
		in.defaultReadObject();
		command.run();
	}
}
```


## How Records Fixes Serialization


## JEP 290 ???

**More resources**

* JEP 395

* JEP 290

* Brian Goetz on Serialization

* Snyk Article on serialization


 