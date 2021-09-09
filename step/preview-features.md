# Java SE Preview Features

## Overview


This  5-minutes lab will give you an introduction to the Java SE **Preview Feature** mechanism.

The Preview Feature mechanism enables to add non-final, fully specified, and implemented features into the Java platform. The goal is to allow developers to use those non-final features, to gather feedback, and to make necessary changes if needed before those features are made final and permanent into the Java platform.

⚠️ **Preview Features** should not be confused with **Experimental Features** or with **Incubator Modules**. For more information, please check this [article](https://blogs.oracle.com/javamagazine/the-role-of-previews-in-java-14-java-15-java-16-and-beyond). 


## Hello Preview 


1. Create a sealed classes hierarchy

In a new directory, create a simple _Test.java_ class.

```nohighlight
<copy>
nano Test.java
</copy>
```

```java
<copy>
class Test {
   public static void main(String ... args) {
      var result = switch ("Lorem Ipsum") {
         case null         -> "Oops";
         case "Foo", "Bar" -> "Great";
         default           -> "Ok!";
      };

      System.out.println(result);
   }
}
</copy>
```

⚠️ This example uses the Pattern Matching for Switch (see Step 10). A this stage, its sole purpose is to introduce the concept of Preview Feature.

2. Compile it

```nohighlight
<copy>
javac Test.java
</copy>
```

You will get an error saying that _"...null in switch cases is a preview feature and is disabled by default."_, the message also suggests to  _"(use --enable-preview to enable null in switch cases)"_.

This error simply informs you that you are trying to use a preview feature of Java 17, and that those are disabled by default. To use preview features, you need to explicitly enable them, at compile-time, using the `--enable-preview` flag. Note that, you also need to confirm to the Java compiler which version of the Preview Feature you are using (ex. using the `--release` flag). 

```nohighlight
<copy>
javac --enable-preview --release 17 Test.java
</copy>
```

Those 2 flags are enforcing a safeguard mechanism that informs you that non-permanent features are used, and hence those might change in a future Java release.

The compilation now succeeds. Notice that you are still warned that preview features are used in the code.

To run code that uses Preview Feature, you would face the same safeguard as Preview Features are also disabled at runtime! To be used, they should be explicitly enabled using the `--enable-preview` flag. The difference is that at runtime, you don't need to use a flag to confirm the version that you are using.

```nohighlight
java --enable-preview Test
```

## Preview Features & Helidon

Likewise, to use a Preview Feature in Helidon, those should be enabled at both compile-time and runtime.

#### Compile-time configuration

In an Helidon project's `pom.xml`, configure, in the `<plugins>` section, the Java compiler plugin to Java 17 **and** to enable Preview Features.

```
<copy>
<plugin>
   <groupId>org.apache.maven.plugins</groupId>
   <artifactId>maven-compiler-plugin</artifactId>
   <version>3.8.0</version>
   <configuration>
       <release>17</release>
       <compilerArgs>--enable-preview</compilerArgs>
   </configuration>
</plugin>
</copy>
```



#### Runtime configuration

To run the application, use the following command.

```nohighlight
<copy>
java --enable-preview -jar target/myapp.jar
</copy>
```

Similarly, to use Preview Features via the Helidon CLI 'devloop', you need to pass the same `--enable-preview` argument to the JVM running the application:

```nohighlight
<copy>
helidon dev --app-jvm-args "--enable-preview"
</copy>
```

⚠️ If during this lab, Helidon hangs while starting ('devloop'), double-check that you have effectively enabled preview features! 


## Wrap-up

In this section, you have used Pattern Matching for switch, a **Preview Feature** of Java 17. That particular feature will be covered in an upcoming section. Moreover, you have seen how to enable Preview Features in Helidon applications.

In summary, the **Preview Feature** mechanism:
* allows introducing non-final features into the Java platform (ex. Language Feature)
* allow developers to use those and provide feedback
* enables Oracle to gather that feedback and make changes if needed
* Preview Features are disabled by default, they should explicitly be enabled at both compile-time and runtime
* a given Preview Feature is specific to a specific Java version


 
