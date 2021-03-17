# Lab 12: jlink

## Overview

This 5-minute optional lab will show you how **`jlink`** can greatly improve the size of a Java runtime image.

In a nutshell, `jlink` is a tool that can create custom Java runtimes that include only the modules required by an application. Reducing the numbers of modules will reduce the overall size of that custom Java runtime image, a concern especially important when a Java application runs within a container. `jlink` has been part of the JDK since JDK 9.

## Using `jlink`


First, create a minimal application and compile it.


```nohighlight
mkdir ~/jlink-test && cd ~/jlink-test
nano Test.java
javac Test.java
```

```
<copy>
class Test {
   public static void main(String args[]) {
     System.out.println("Hello jlink!");
   }
}
</copy>
```

You then need to know which module(s) this application requires to run. For that, you will use another JDK tool, `jdeps`.

```nohighlight
<copy>
jdeps Test.class
</copy>
```

You can see that in this case only the `java.base` module is required. Using this information, you can create a custom runtime by passing to `jlink` the target location of the custom runtime and the list of module(s) to include in it.

```nohighlight
<copy>
jlink --output custom-runtime --add-modules java.base
</copy>
```

You can now check the size of the generated Java runtime image.

```nohighlight
<copy>
du -chs custom-runtime
</copy>
```

This small (<50MB!) custom Java runtime is limited but it can, nevertheless, runs any application that only requires the `java.base` module such as the example above.


## Using `jlink` with Helidon applications 


The previous example could not be more simple! As such, it does not reflect the reality as any real application will be way more complex than a simple "Hello World" class. The challenge when creating a custom Java runtime image mostly comes from the dependencies that the application is using as you have to figure out which modules are used, and hence required in your custom runtime image. As you saw, a tool such as `jdpes` can help to identify those modules. The good news is that more and more Java frameworks support `jlink` out-of-the-box! As a developer, you don't have to worry about making sure that you have correctly identified all the modules required by your application and its dependencies as the framework tooling will often take care of that!

To create a jlink based custom Java runtime image, Helidon is using a Maven profile. To use it, simply issue the following Maven command from the project directory.

```nohighlight
<copy>
cd ~/odl-java-hol
mvn package -Pjlink-image -Djlink.image.defaultJvmOptions="--enable-preview"
</copy>
```

![](./images/lab11-1.png " ")

The result are impressive as the total size (the Java runtime, the application and its dependcies) went from ~320MB to ~80MB, a ~75% gain!

💡 The Helidon jlink Maven profile also creates, by default, a CDS (Class Data Sharing) archive. CDS is a JDK feature that helps reduce the startup time and memory footprints of Java applications. Check [here]((https://docs.oracle.com/en/java/javase/16/vm/class-data-sharing.html#GUID-7EAA3411-8CF0-4D19-BD05-DF5E1780AA91) from additional details.

A convinent startup script is also created in the process, it handles for example the JVM parameters such as `--enable-preview` in this particular case. To run the application with its custom Java runtime image, simply invoke this script.

```nohighlight
<copy>
target/conference-app/bin/start
</copy>
```

To get additional details, just use its help.
```nohighlight
<copy>
conference-app/bin/start --help
</copy>
```

As you can see, using `jlink` with Helidon is simple and straight forward!


## Wrap-up

**`jlink`** is a tool that can assemble and optimize a set of modules and their dependencies to create a custom, i.e. optimized, run-time Java image. Optimizing the size of a Java runtime is critical when an Java application is embedded with its Java runtime into a container. Reducing the overall container image size will improve the startup time of this container. Moreover, smaller container leads to better resource utilization of the container platform.  It should be mentioned that to use `jlink` the application does not need to be modularized! Moreover, `jlink` is available since JDK 9 so there is no valid reason to not leverage `jlink` today!

**More ressources**
* [The `jlink` Command](https://docs.oracle.com/en/java/javase/16/docs/specs/man/jlink.html)
* [The `jdeps`command](https://docs.oracle.com/en/java/javase/16/docs/specs/man/jdeps.html)
* [JEP 282: `jlink`](https://openjdk.java.net/jeps/282)
* [Helidon SE — Custom Runtime Images with `jlink`](https://helidon.io/docs/v2/#/se/guides/37_jlink_image)
* [Helidon Maven Plugin](https://github.com/oracle/helidon-build-tools/tree/master/helidon-maven-plugin#goal-jlink-image)
* [Class Data Sharing](https://docs.oracle.com/en/java/javase/16/vm/class-data-sharing.html#GUID-7EAA3411-8CF0-4D19-BD05-DF5E1780AA91)



 