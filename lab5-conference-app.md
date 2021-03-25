# Lab 5: The Sample Application

<div style="display: none;"><span><img src="https://129.146.125.59:8080/p/odl-16-lab/5"></span></div>

## Overview 


In this 5-minutes lab, you will clone and build an Helidon application that will be used to test some of the recent Java features.

This simple application is themed around 'conferences', it provides a simple REST endpoint that lists speakers and a basic web user interface.

⚠️ For the sake of brevity and clarity, this application takes some shortcuts and does not necessarily implement all the best practices recommended for an application that would go into production. Typically, you would have to think about concerns such as security, synchronization, testing, data validation, availability, scaling, etc. none of which are relevant in the context of today's lab!


## The Sample Application


The application source is hosted on GitHub, just clone its repository:

```nohighlight
cd ~
git clone https://github.com/delabassee/odl-java-hol.git
cd odl-java-hol
```


This repository has several branches (`git branch -a`).&nbsp;

* `lab5` : starting point
* `lab6` : lab 6 starting point, including the lab 5 solution
* `lab7` : lab 7 starting point, including the lab 5 and 6 solution
* `lab8` : lab 8 starting point, including the lab 5 to 7 solutions
* `lab9` : lab 9 starting point, including the lab 5 to 8 solutions
* `lab10` : lab 10 starting point, including the lab 5 to 9 solutions
* `lab11` : all solutions from lab 5 to 10 included

💡 Lab 11, 12, and 13 are not based on the sample application.

Switch to the starting point

```nohighlight
<copy>
git checkout lab5
</copy>
```

Update the project's `pom.xml` to enable Preview Features as described in the previous section.

## Build and test the application

By now, you should know how to build and test an Helidon application. 

Either using Maven:

```
mvn package

# for an application that is not using any preview features
java -jar target/conference-app.jar

# for an application that is using preview features
java --enable-preview -jar target/conference-app.jar
```

or using the Helidon CLI devloop:

```
# for an application that is not using any preview features
helidon dev

# for an application that is using preview features
helidon dev --app-jvm-args "--enable-preview"
```

The sample application offers a simple Web user interface (http://{public-ip}:8080/public/), and exposes mutliple REST endpoints to get speaker-related information.

* http://{public-ip}:8080/speakers ➡ Get all speakers
* http://{public-ip}:8080/speakers/company/{company} ➡ Get speakers for a given company
* http://{public-ip}:8080/speakers/lastname/{name} ➡ Get speaker by its lastname
* http://{public-ip}:8080/speakers/track/{track} ➡ Get speakers for a given track
* http://{public-ip}:8080/speakers/{id} ➡ Get speaker details for a given id

Once the application is running, you can test it. 

* http://{public_ip}:8080/speakers/lastname/goetz
* http://{public_ip}:8080/speakers/company/oracle
* http://{public_ip}:8080/speakers/track/java

## Lab Navigation & Tips

Here are a few tips that might be useful in the course of this Lab.

* If the left navigation bar is too intrusive, just hide it temporarily using the top-left hamburger icon.

* To switch between branches, use `git branch checkout {target-branch}`, ex. `git checkout -f lab10`

* To list branches: `git branch -a`

* Any given branch contains the solutions of the preceding exercises. If you are lost just checkout the n+1 branch and check your code.

* You can also browse branches' content directly on [GitHub](https://github.com/delabassee/odl-java-hol/branches)

* During this lab, you will only do simple Java coding so you won't use a Java IDE. Instead, you will use the versatile `nano` text editor. Here are some of its important key shortcuts.

	* <kbd>[Control]</kbd> + <kbd>[x]</kbd> : Exit
	* <kbd>[Control]</kbd> + <kbd>[o]</kbd> : Save
	* <kbd>[Control]</kbd> + <kbd>[k]</kbd> : Delete Line
	* <kbd>[Control]</kbd> + <kbd>[g]</kbd> : Help
	* <kbd>[Control]</kbd> + <kbd>[y]</kbd> : Page Up
	* <kbd>[Control]</kbd> + <kbd>[v]</kbd> : Page Down

* It is recommended to use Firefox to test REST endpoints as it renders nicely the returned JSON payload. If you are comfortable with CLI, you can also use `curl` in combination with `jq` to format JSON responses.

* If you get errors while using Helidon's devloop, you might want to re-build the project using Maven to get additional details on those errors.

* For brevity, packages will sometimes be omitted from code snippets, they are obviously required. If you are not sure, simply check the solution.

* To view files, you can use the '`bat`' tool as it offers syntax highlighting.

* Any change made to a `pom.xml` will not be detected by Helidon `devloop`, you need to restart it.

* If you are using the Helidon `devloop`, make sure to enable preview features!

```nohighlight
<copy>
helidon dev --app-jvm-args "--enable-preview"
</copy>
```
