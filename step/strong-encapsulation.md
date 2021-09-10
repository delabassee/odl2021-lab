# Strong Encapsulation

## Overview

In this 10-minute lab, you will learn about the importance and evolution of strong encapsulation, which APIs it encapsulates, and how to configure exceptions if needed.

## What is strong encapsulation about?

In many respects, the OpenJDK code base is similar to any other software project and one constant is refactoring.
Code is changed, moved around, removed, etc. to keep the code base clean and maintainable.
Not all code of course:
The public API, the contract with Java's users, is extremely stable.

As you can see, the distinction between public API and internal code is paramount, to the JDK developers but also to you.
You need to be sure that your project, meaning your code _and_ your dependencies, don't rely on internals that can change in any minor JDK update, causing surprising and unnecessary work.
Worse, such dependencies might block you from updating the JDK.
At the same time, you might be in a situation where an internal API provides unique capabilities without which your project couldn't compete.

💡 Together, this means that a mechanism that locks internal APIs away by default but allows you to unlock specific ones for specific use cases is essential.
That is what strong encapsulation, provided by the Java module system, does.
All JDK code is split into modules and modules can export packages.
Only public types (and their public members) in exported packages are accessible to code outside the module.
Everything else is considered internal and inaccessible during compilation _and_ at run time, i.e. even reflection can no longer break into internal APIs.

## What are internal APIs?

So which JDK APIs are internal?
To answer that, we need to look at three namespaces:

First `sun.*`.
Almost all such packages are internal, but this is where the exceptions come in:
These are the `sun.misc` and `sun.reflect` packages, which are exported and opened by the module _jdk.unsupported_ because they provide functionality that is critical to a number of projects and doesn't have feasible alternatives within or outside the JDK.
All other `sun.*` packages are internal.

Second is `com.sun.*`, which is more complicated.
The entire namespace is JDK-specific, meaning it's not part of Java's standard API, and some JDKs may not contain it.
Around 90% of it are non-exported packages and they are internal.
The remaining 10% are packages exported by _jdk.*_ modules and they're supported for use outside the JDK.
That means they are evolved with a similar regard for compatibility as standardized APIs.
[Here's a list](https://cr.openjdk.java.net/~mr/jigsaw/jdk8-packages-strongly-encapsulated) of internal vs exported packages.

The third one is `java.*`.
Of course, these packages make up the public API but that only extends to public members of public classes.
Less visible classes and members are just as internal as what we discussed so far.

💡 In summary, use `java.*`, avoid `sun.*`, be careful with `com.sun.*`.

## Experiments with strong encapsulation

To experiment with strong encapsulation, let's create a simple class...

```shell
<copy>
nano Internal.java
</copy>
```

...that uses a class from a public API:

```java
<copy>
public class Internal {

	public static void main(String[] args) {
		System.out.println(java.util.List.class.getSimpleName());
	}

}
</copy>
```

Since it's a single class, you can run it straight away without explicit compilation:

```shell
<copy>
java Internal.java
</copy>
```

This should run successfully and print "List".

Next, let's mix in one of those exceptions that are available for compatibility reasons:

```java
<copy>
// add to `main` method
System.out.println(sun.misc.Unsafe.class.getSimpleName());
</copy>
```

You will still be able to run this straight away, printing "List" and "Unsafe".

Now let's use an internal class that is not accessible:

```java
<copy>
// add to `main` method
System.out.println(sun.util.PreHashedMap.class.getSimpleName());
</copy>
```

If you try to run this as before, you get a compile error (the `java` command compiles in memory):

```shell
Internal.java:8: error: package sun.util is not visible
                System.out.println(sun.util.PreHashedMap.class.getSimpleName());
                                      ^
  (package sun.util is declared in module java.base, which does not export it)
1 error
error: compilation failed
```

The error message is pretty clear:
The package `sun.util` belongs to the module _java.base_ and because that doesn't export it, it is considered internal and thus inaccessible.
If you absolutely need to access it, though, you can with the command line option `--add-exports`:

```shell
<copy>
java --add-exports java.base/sun.util=ALL-UNNAMED Internal.java
</copy>
```

This makes the module _java.base_ export the `sun.util` package to code on the class path and thus the program runs successfully and prints "List", "Unsafe", and "PreHashedMap".

## Strong Encapsulation in Practice

There are two command line options that let you work around strong encapsulation:

* `--add-exports` makes public types and members in the exported packages accessible at compile or run time
* `--add-opens` makes all types and their members in the opened package accessible at run time (for reflection)

When applying `--add-exports` during compilation, it must be applied again when running the app and of course `--add-opens` can only be applied at run time.
That means that whatever code (yours or your dependencies) needs access to JDK internals, the exceptions need to be configured when launching the app.
That gives the app's owner full transparency into these issues and allows them to assess the situation and either change the code/dependency or knowingly accept the maintainability hazard that comes from using internal APIs.

Strong encapsulation is in effect around all explicit modules.
That includes the entire JDK, which is fully modularized, but potentially also your code and your dependencies, should they come as modular JARs (i.e. JARs with a `module-info.class`) that you place on the module path.
In that case, everything said so far applies to these modules as well.

## Java 17

Strong encapsulation is a corner stone of the module system, which was introduced in Java 9.
But for compatibility reasons, code from the class path could still access internal JDK APIs.
This was managed with the command line option `--illegal-access`, which had the default value `permit` in JDK 9 to 15.
JDK 16 changed that default to `deny` and 17 deactivates the option entirely.

💡 From 17 on, only `--add-exports` and `--add-opens` give access to internal APIs.

## Wrap-up

In this exercise you have learned that:

* it is important not to (accidentally) use internal APIs because they can change without warning
* the module system facilitates that with strong encapsulation for modules
* only public types and members in exported packages are accessible outside a module
* for the JDK modules that's the `java.*` packages
* other types and members are inaccessible during compilation and at run time
* exceptions can be created with `--add-exports` (for static dependencies) and `--add-opens` (for reflective access)

Check out the following links to learn more about the module system and strong encapsulation:

* [JEP 403: Strongly Encapsulate JDK Internals](https://openjdk.java.net/jeps/403)
* [_Java's Steady March Towards Strong Encapsulation_ with Alan Bateman - Inside Java Podcast 18](https://www.youtube.com/watch?v=dRX_stwoOgo)
* [Code-First Java Module System Tutorial](https://nipafx.dev/java-module-system-tutorial/)
