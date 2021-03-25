# Lab 6: Text Blocks

## Overview

In this 10-minutes lab, you will use **Text Blocks**, a new Java feature.

A text block is a multi-line string literal that avoids the need for most escape sequences. It automatically formats the string predictably, and gives the developer control over format when desired. The Text Blocks feature has been previewed in Java 13 and Java 14 and is a permanent feature since Java 15.

## Text Blocks in more details

Text blocks enable developers to easily embed string literals spanning multiple lines (ex. structured languages such as JSON, XML, etc.) into Java source code while preserving the formatting of the string literal and also keeping the Java code readable.

For example, imagine that you have to embed in your Java code an HTML snippet that displays '_All I want to see is a "_'. Notice the double quotes at the end! Before Text Blocks, you would write something like this.

```
var element = "<p id=\"p1\">All I want to see is a \"</p>";
```

Notice that the HTML _id_ element attribute above requires its value to be enclosed in double-quotes, so those double quotes need to be escaped in the Java code. 

Things get worst if you want to preserve the readability and the formatting of the snippet. For example, to embed the following basic HTML list in Java code:

```
<ul id="test">
   <li><a href="https://abc.org/">ABC</a></li>
   <li><a href="https://xyz.org/">XYZ</a></li>
</ul>
```
you would have to write something like this:

```
var test = "<ul id=\"test\">\n" +
           "   <li><a href=\"https://abc.org/\">ABC</a></li>\n" +
           "   <li><a href=\"https://xyz.org/\">XYZ</a></li>\n" +
           "</ul>";
```


Not only is such code hard to write, and hence error-prone, but it is also hard to read, and hence hard to maintain.

Thanks to Text Blocks, it is now a lot easier! A text block is simply a [String](https://docs.oracle.com/en/java/javase/14/docs/api/java.base/java/lang/String.html) literal that can span over multiple lines. A Text Block is delimited with the new **triple-quote delimiter** (`"""`) instead of the traditional double-quote delimiter (`"`).

```
var test = """
        <ul id="test">
            <li><a href="https://abc.org/">ABC</a></li>
            <li><a href="https://xyz.org/">XYZ</a></li>
        </ul>
        """; 
```

As you can see from the example below
- a Text Block starts and ends with a triple-quote delimiter (`"""`)
- the line after the opening triple-quote delimiter should be empty or blank
- any double-quote embedded in the string shouldn't be escaped (!)
- the string literal can span multiple lines without any additional escaping (`\n`)
- the original formatting is preserved, it is controlled by the closing delimiter
- and last but not least, the code readability is greatly improved!

To preserve formatting while improving code readability, Text Blocks differentiate __incidental white spaces__, from __essential white spaces__. Incidental white spaces are used to improve source code readability, they will be stripped away automatically by the Java compiler.

The following code example is identical to the previous one but it doesn't use incidental white spaces. Incidental whitespace can be used to improve the readability of the source code, but doesn't impact how the Java compiler interprets the value of the Text Block.


```
var test = """
<ul id="test">
   <li><a href="https://abc.org/">ABC</a></li>
   <li><a href="https://xyz.org/">XYZ</a></li>
</ul>
"""; 
```
The triple-quote closing delimiter defines how incidental white spaces are handled. Please check the resources at the end of the exercise for more details on those rules.

## Add Text Blocks support

In the `Main.java` class, you can notice that the application uses Helidon's Web Server [Static Content support](https://helidon.io/docs/v2/#/se/webserver/06_static-content-support) to expose some static content.

```nohighlight
<copy>
nano src/main/java/conference/Main.java
</copy>
```

```
Routing.builder()
    .register("/public", 
        StaticContentSupport.builder("public").welcomeFileName("index.html"))
...
```

This static content is exposed under the `/public` path, and is served from the `/public` directory in the `/resources` directory of the application. `index.html` is the default file served.

```nohighlight
<copy>
bat src/main/resources/public/index.html
</copy>
```

Run the application and access, from your browser, the `/public` url, ex. `http://{public-ip}:8080/public`. You should get a basic UI to list speakers.
![](./images/lab5-1.png " ") 

If you try to access the root path (`http://{public-ip}:8080/`), you will get an error as there is no handler defined to handle this path. 


<img src="./images/lab5-2.png" alt="error message" width=50%/>


To fix this, any HTTP request to the `/` path should be forwarded to the `/public` path.


**1. Define an HTML snippet to trigger a client-side forward**

In the `createRouting` method, define a Text Block that embeds some HTML to trigger a client-side forward to the `/public` path.


```nohighlight
<copy>
nano src/main/java/conference/Main.java
</copy>
```

```
<copy>
var snippet = 
    """
    <html>
       <title>Almost there</title>
       <body>
          You are being redirected...
          <meta http-equiv="refresh" content="0; url=/public/" />
       </body>
    </html>
    """;
</copy>	
```

**2. Update the application routing**

Update the application routings to serve that HTML snippet whenever someone hits the root `/` path.


```
return Routing.builder()
    <copy>.get("/", (req, res) -> { res.send(snippet); } )</copy>
    .register("/public", 
        StaticContentSupport.builder("public")
             .welcomeFileName("index.html"))
    .register("/speakers", speakerService)
    .build();

```

Let's check the snippet that was just added.

```
   …
   .get("/", (req, res) -> { res.send(snippet); } )
   …
```

It defines a [handler](https://helidon.io/docs/v2/apidocs/io.helidon.webserver/io/helidon/webserver/Handler.html) for any HTTP GET requests under the `/` path.  It has one method ([accept](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/function/BiConsumer.html?is-external=true#accept(T,U))) that takes a `BiConsumer<ServerRequest, ServerResponse>` to handle the request. In this case, we pass it a Lambda expression that sends the HTML snippet that will trigger the forward on the client-side.



**3. Test the application**

Compile, and access the application from a browser, `http://{public-ip}:8080/`

<img src="./images/lab5-3.png" alt="raw html" width=60%/>

You can notice that the HTML formatting has been preserved which is good but that is not really what we expected as the HTML is not rendered in the browser! To understand the issue, use `curl` to get more details on the HTTP request/response.


```
> <copy>curl -v http://{public-ip}:8080</copy>
...
< HTTP/1.1 200 OK
< Content-Type: text/plain
...
```

You can see that the content type of the HTTP response is set to `text/plain` and not `text/html`! That explains why the HTML is not parsed and rendered by the browser. This can be easily fixed by correctly setting the MediaType header before sending the response. 

```
…
<copy>
.get("/", (req, res) -> {
       res.headers().contentType(MediaType.TEXT_HTML);
       res.send(snippet);
    })
</copy>
…
```

💡 Make sure to update the imports to include `import io.helidon.common.http.MediaType`

Now when you access the root path, your browser should be automatically redirected to the simple HTML UI.


**4. Understand how Text Block works**

Remove the `res.headers().contentType(MediaType.TEXT_HTML);` line to bypass HTML rendering on the client side. Try mutilple variations by adjusting the closing delimiter to understand how the incidental spaces are handled. 

💡 You might want to check the resources suggested in the next section to understand some behaviors.


```
<copy>
message = """
             Does this one work?
             """;
</copy>
```

```
<copy>
message = """
          And
		what
	  about
	this?
    """;
</copy>
```

```
<copy>
message = """
          A single line Text Block?""";
</copy>
```

```
<copy>
message = """And what about this?""";
</copy>
```


## Wrap-up


In this exercise, you have used Text Blocks, a standard Java 15 feature, to easily embed HTML into Java code. 

Simply put, Text Blocks enable developers to easily embed string literals spanning multiple lines into Java source code while preserving the original formatting but also Java code readability. Text blocks are handy to deal with structured languages (ex. XML, JSON, etc.) without having to worry about escaping special characters (ex. new line, double quotes) nor altering the original formatting. Finally, the power of the [String API](https://docs.oracle.com/en/java/javase/16/docs/api/java.base/java/lang/String.html) remains at our disposal with Text Blocks.

Check the following resources for more details on Text Blocks.

* [JEP 378: Text Blocks](https://openjdk.java.net/jeps/378)
* [Programmer's Guide To Text Blocks](https://inside.java/2019/08/06/text-blocks-guide/)
* [Java Feature Spotlight: Text Blocks](https://inside.java/2020/05/01/spotlighttextblocks/)

<div style="display: none;"><span><img src="https://129.146.125.59:8080/p/odl-16-lab/6"></span></div>




