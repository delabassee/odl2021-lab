# jpackage

## Overview

This 10-minute optional lab will introduce you to **`jpackage`**, a tool that has started to incubate in JDK 14, and has been a production-ready feature in JDK 16.

The `jpackage` tool creates self-contained, platform-specific, Java application bundles. These bundles provide a simple solution to distribute your Java applications as end-users will rely on the platform's usual approaches to install them (ex. double click on a .exe file on Windows, ...), upgrade them, and un-install them.

`jpackage` supports the following platform-specific bundle types:

* Windows: **msi** and **exe**
* Linux: **deb** and **rpm**
* macOS: **pkg** and **dmg**

In a nutshell, an application bundle is composed of the Java applications itself with its dependencies, and a Java runtime.

💡 To generate an application bundle, `jpackage` relies on platform-specific tools, ex. it uses `rpm-build` to generate packages for rpm compatible Linux distribution such as Oracle Enterprise Linux.


## Prepare your application


First, create a minimal Helidon application, and buil it.


```
mkdir ~/jpackage-test && cd ~/jpackage-test
helidon init
# keep the default suggested values…

# and build the application
cd java-devlive
mvn package
```


Next, you need to isolate the bits that should be part of your bundle, i.e. the application and its dependencies.

```
mkdir to-bundle
# Copy the application jar
cp target/demo.jar to-bundle

# and the application's dependencies
cp -r target/libs/ to-bundle/libs
```


## Using `jpackage` to create an application bundle

We can now create, using `jpackage` an application bundle that will allow our end-user to easily install it.

```
<copy>
jpackage --type rpm \
         --input to-bundle \
         --name simple-app \
         --main-jar demo.jar \
         --main-class com.devlive.Main \
         --vendor 'HoL & Co' \
         --verbose \
		 --app-version 1.0 \
         --description 'simple jpackage test'
</copy>
```

Producing the bundle will take some times. In the meantime, let's look at the used parameters...

* `--type`, `-t` : The type of package to create
* `--input`, `-i` : Path of the input directory that contains the files to be packaged
* `--name`, `-n` : Name of the application and/or package
* `--main-jar` : The main JAR of the application
* `--main-class` : Qualified name of the application main class to execute
* `--vendor` : Vendor of the application (optional) 
* `--verbose` : Display details while the bundle is being produced (optional) 
* `--app-version` : Version of the application and/or package (optional) 
* `--description` : Short description of the application (optional) 

After ~75 seconds, your `rpm` bundle will be generated.

```nohighlight
<copy>
ls -la simple-app-1.0-1.x86_64.rpm
</copy>
```

## Use the application bundle


You can now use `rpm` as usual...

* Install the application 

```nohighlight
<copy>
sudo rpm -i simple-app-1.0-1.x86_64.rpm
</copy>
```

* Check that your application is installed  

```nohighlight
<copy>
rpm -qa | grep simple-app
</copy>
```

* Get additional details about your installed application 

```nohighlight
<copy>
rpm -qi simple-app-1.0-1.x86_64
</copy>
```

```
Name        : simple-app
Version     : 1.0
Release     : 1
Architecture: x86_64
Install Date: Mon Feb  8 10:41:32 2021
Group       : Unspecified
Size        : 143231221
License     : Unknown
Signature   : (none)
Source RPM  : simple-app-1.0-1.src.rpm
Build Date  : Mon Feb  8 10:36:20 2021
Build Host  : instance-20210205-1634.sub02010939050.holvcn.oraclevcn.com
Relocations : /opt
Vendor      : HoL & Co
Summary     : simple-app
Description : simple jpackage test
```

* Invoke the application launcher

```nohighlight
<copy>
/opt/simple-app/bin/simple-app
</copy>
```

* Remove the application

```nohighlight
<copy>
sudo rpm -e simple-app-1.0-1.x86_64
</copy>
```


## Wrap-up

The **`jpackage`** tool creates self-contained, platform-specific, Java application bundles.


This is a trivial example. `jpackage` offers many more options such as the ability to create a bundle based on a modularized Java application, let the user customize the installation (ex. where to install the application, ...), etc. For more details, please check the [Packaging Tool User's Guide](https://docs.oracle.com/en/java/javase/16/jpackage/packaging-overview.html#GUID-C1027043-587D-418D-8188-EF8F44A4C06A) and [JEP 392: Packaging Tool](https://openjdk.java.net/jeps/392).

💡 It is also possible to pass argument(s) to execute the application, ex. to enable the application to use preview-features. 

One thing you can notice is that the size of the final bundle is relatively big, i.e. ~135 MB for this example. This is because the bundle includes the Java application, its dependencies, but also a Java runtime! That means that to run the application, the end-user will only need the application bundle and nothing else! By default, `jpackage` will pick up the machine's Java runtime but it is possible to specify a different Java runtime. For example, you can use `jpackage` (JDK 16) to create a bundle that will include a JDK 11 based Java runtime. To work around the Java runtime image size issue, `jpackage` can leverage `jlink` to create a custom Java runtime that will only include the modules required to run the application. This will greatly reduce the size of the final bundle. The next Lab will give an overview of `jlink`.






 







