# Wrap-up

<div style="display: none;"><span><img src="https://billy.delabassee.com:8080/p/odl-16-lab/end"></span></div>

In this hands-on lab, you have used Helidon on the Oracle Cloud to explore some new Java features.


Given the time constraints and the remote aspect, we did not have the opportunity to necessarily go very deep and cover all the details. Moreover, we had to make a selection and skip some other important new features. For example, we didn't discuss the different Project Panama related APIs present in JDK 16. Covering those at a very high-level would require another 2h at the very minimum!

In a nutshell, there are 3 Panama related APIs in JDK 16:
- The [Vector API](https://openjdk.java.net/jeps/338), which enables to easily express vector operations that will be compiled at runtime to be executed on a CPU Vector unit.
- The [Foreign Memory Access API](https://openjdk.java.net/jeps/393), which enables to access safely and efficiently native memory from Java.
- The [Foreign Linker API](https://openjdk.java.net/jeps/389), which allows invoking safely native code from Java code.

We haven't discussed any of the underlying HotSpot improvements such as, for example, the new [ZGC Concurrent Thread-Stack Processing](https://openjdk.java.net/jeps/376) which enables sub-milliseconds ZGC pause time. We didn't cover the fact that OpenJDK is now entirely developed using Git on GitHub (see [http://github.com/openjdk/jdk](http://github.com/openjdk/jdk)). We haven't touched any of the security related enhancements, etc. Each Java release comes with a set of new features, with a lot of various enhancements, and Java 16 is not an exception! 

Today, you have used Helidon simply because it's small, fast, and un-intrusive but there was nothing specific to Helidon. All the features discussed in this lab will work with any frameworks or libraries that support Java 16 (ex. Micronaut).

We hope you have learned something today. The call to action is to download Java 16 right after this lab, and continue your exploration!

For more information on Java 16, and on the Java platform in general, we encourage you to regularly check [Inside Java](https://inside.java), and to follow [@Java](https://twitter.com/java) on Twitter. Moreover, you might also want to listen to the [Inside Java Podcast](https://inside.java/podcast).


PS: Do not forget that you now have access to the [Oracle Cloud Free Tier (Always Free)](https://www.oracle.com/cloud/free/)!


## Resources


* [Download JDK 16](https://jdk.java.net/16/)
* [JDK 16 Documentation](https://docs.oracle.com/en/java/javase/16/)
* [JDK 16 Release Notes](https://jdk.java.net/16/release-notes)
* [Inside Java Podcast](https://inside.java/podcast)
* [Helidon](https://helidon.io/#/)
* [cloud.oracle.com](https://cloud.oracle.com)
* [Oracle Cloud Infrastructure Documentation](https://docs.oracle.com/en-us/iaas/Content/home.htm)







