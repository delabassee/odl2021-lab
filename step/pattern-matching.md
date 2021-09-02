# Pattern Matching for instanceof

## Overview

In this 10-minutes lab, you will get some hands-on experience with the **pattern Matching for instanceof** feature previewed in Java 14 and Java 15, and made final and permanent in Java 16.

Pattern matching allows common logic in a program, namely the conditional extraction of components from objects, to be expressed more concisely and safely. This new feature enhances the Java programming language with an initial form of pattern matching, i.e. **pattern matching for the instanceof operator**. 


## Using 'pattern matching for instanceof'

All Java programmers are familiar with the "instanceof-and-cast" idiom.

```
if (obj instanceof String) {
    String s = (String) obj;  // grr!
	System.out.println(s);
    …
}
```

1. a test: is `obj` a `String`?
2. a conversion: casting `obj` to `String`
3. the declaration of a new local variable to use the object value: `s`

Although straightforward and well understood, this idiom is suboptimal and tedious! In Java 16, the `instanceof` operator has been extended to take a **type pattern** instead of just a type. A type pattern consists of a **predicate that specifies a type**, along with a **binding variable**, ex. `String s`. Such a pattern is easy to grasp: if `obj` is an instance of type `String`, then its value is assigned to the new `s` variable of type String.

```
// pattern matching for instanceof
if (obj instanceof String s) {
    System.out.println(s);  // yeah!
    …
}
```



💡 Make sure to checkout the lab10 branch as it introduces 2 new classes to the project: `AgendaRepository.java` and `AgendaService.java`

```nohighlight
<copy>
git checkout -f lab10
</copy>
```

Check those 2 new classes. `AgendaService.java` introduces a new "/sessions" endpoint that returns the details of the sessions. The sessions are stored in `AgendaRepository.java` using a simple `List<Session>`. The `Session` type has been introduced in Lab 8, it is a sealed abstract class that can only be extended by a given set of classes (check Lab 8 for details).

Build and test the application, `curl {public_ip}:8080/sessions`

![](../images/lab10-1.png " ")



Let's pretend that the displayed details should vary based on the session type.

Add the following `getSessionDetails` method to the "AgendaService".

```nohighlight
<copy>
nano src/main/java/conference/AgendaService.java
</copy>
```

```
<copy>
private void getSessionDetails(final ServerRequest request, final ServerResponse response) {
   LOGGER.fine("getSessionDetails");

   var sessionId = request.path().param("sessionId").trim();

   Optional<Session> session = sessions.getBySessionId(sessionId);

   if (session.isPresent()) {

      record SessionDetail(String title, String speaker, String location, String type) {}

      var detail = "speaker TBC!";
      var s = session.get();

      if (s instanceof Keynote) {

         Keynote k = (Keynote) s;
         var ks = speakers.getById(k.getKeynoteSpeaker());

            if (ks.isPresent()) {
               var spk = ks.get();
               detail = spk.firstName() + " " + spk.lastName() + " (" + spk.company() + ")";
            } else detail = "Keynote speaker to be announced!";

            var keynote = new SessionDetail("Keynote: " + k.getTitle(), detail, "Virtual hall", "General session");
            response.send(keynote);
        }
		
		else if (s instanceof Lecture) {

           Lecture l = (Lecture) s;
           var speaker = speakers.getById(l.getSpeaker());

           if (speaker.isPresent()) {
              var spk = speaker.get();
              detail = spk.firstName() + " " + spk.lastName() + " (" + spk.company() + ")";
           }

           var lecture = new SessionDetail(l.getTitle(), detail, String.valueOf(l.getVirtualRoom()), "Conference session");
           response.send(lecture);
        }
		
		else if (s instanceof Lab) {

           Lab l = (Lab) s;
           var speaker = speakers.getById(l.getSpeaker());

           if (speaker.isPresent()) {
              var spk = speaker.get();
              detail = spk.firstName() + " " + spk.lastName() + " (" + spk.company() + ")";
           }

           var lab = new SessionDetail(l.getTitle(), detail, String.valueOf(l.getVirtualRoom()), "Hands on Lab");
           response.send(lab);
        }

    } else {
        Util.sendError(response, 400, "SessionId not found : " + sessionId);
    }
}
</copy>
```



Update Helidon's routing to add `getSessionDetails` as a handler for the "/detail" path.

```
@Override
public void update(Routing.Rules rules) {
   rules.get("/", this::getAll);
   <copy>rules.get("/detail/{sessionId}", this::getSessionDetails);</copy>
}
```

Although a bit long, the `getSessionDetails` method is easy to grasp. 


```
Optional<Session> session = sessions.getBySessionId(sessionId);
if (session.isPresent()) {
   record SessionDetail(String title, String speaker, String location, String type) {}
   …
```

It first gets, from the Session list, a given session based on an Id. And if that session is found, a local record is defined…

```
if (s instanceof Keynote) {

   Keynote k = (Keynote) s;

   var speaker = speakers.getById(k.getKeynoteSpeaker());
   if (speaker.isPresent()) {
      var spk = speaker.get();
      detail = spk.firstName() + " " + spk.lastName() + " (" + spk.company() + ")";
   } else speakerDetail = "Keynote speaker to be announced!";

   var keynote = new SessionDetail("Keynote: " + k.getTitle(), detail, "Virtual hall", "General session");
   response.send(keynote);
```

Then, the `instanceof` operator is used to test the actual type of the session object. Based on this type (Keynote, Lecture, or Lab), the logic to create the details will be slightly different, so this logic is repeated multiple times using a "`if … else if …`" chain.

If you zoom on the `instanceof` pattern, you will notice some verbosity as the type, Keynote in this example, is repeated 3 times. First for the actual `instanceof` operator, and then for the casting and the instantiation of the new temporary variable. It's the usual "instanceof-and-cast" idiom.

```
if (s instanceof Keynote) {
   Keynote k = (Keynote) s;
   // do something with k
   …
}
```

The **pattern matching for instanceof** feature simplifies that code by introducing a **type pattern** composed of a **type** and a **binding variable**, `Keynote k` in this example. A binding variable that will be created should the `instanceof` type test be true.

```
if (s instanceof Keynote k) {
   // do something with k
   …
}
// can't use k here!
```

You can now update the code to leverage the `pattern matching for instanceof` features for the 3 Session types.

```
if (s instanceof Keynote k) {
   var speaker = speakers.getById(k.getKeynoteSpeaker());
   …
} else if (s instanceof Lecture l) {
   var speaker = speakers.getById(l.getSpeaker());
   …
} else if (s instanceof Lab l) {
   var speaker = speakers.getById(l.getSpeaker());
   …
}
```

If you test the application, you should get session details varying depending on the session type.

```
curl {public_ip}:8080/sessions/detail/001
curl {public_ip}:8080/sessions/detail/010
```

## Understanding Pattern Matching

Pattern is a general concept than can be used in other places than `instanceof`. If you want to learn more about pattern matching and provide feedback, then you need to visit the [Amber project page](https://openjdk.java.net/projects/amber/). The Amber project page is the one-stop page for everything related to pattern matching in the Java language.

### Pattern Matching for Instanceof

Let us take another look at the expression `s instanceof Keynote k`. It is composed of three elements. 

The _matched target_ is any object of any type. It is the left-hand side operand of the `instanceof` operator, here the `s` variable.

The _pattern_ is a type followed by a variable declaration. It is the right hand-side of the `instanceof`. The type can be a class, an abstract class or an interface.

The result of the matching is a new reference to the _matched target_. This reference is put in the variable that is declared as a part of the pattern. It is created if the _matched target_ matches the _pattern_. This variable has the type you have matched.

The compiler allows you to use the variable `s` wherever it makes sense to use it. The `if` branch is the first scope that comes to mind. It turns out that you can also use this variable in some parts of the `if` statement.

### Pattern Matching for Switch Expressions

Pattern Matching for Switch Expressions is a preview feature of JDK 17. 

Pattern Matching for Switch Expressions uses switch expressions. It allows you to match a _matched target_ to several _patterns_ at once. So far the _patterns_ are _type patterns_, just as in the pattern matching for `instanceof`.

In this case the _matched target_ is the selector expression of the switch. There are several _patterns_ in such a feature; each case of the switch expression is itself a type pattern that follows the syntax described in the previous section.

With this preview feature, you can rewrite the previous code in the following way: 

```
var speaker = 
   switch(s) {
      case Keynote k -> speakers.getById(k.getKeynoteSpeaker());
      case Lecture l -> speakers.getById(l.getSpeaker());
      case Lab l     -> speakers.getById(l.getSpeaker());
   }
```

So far it is not an extension of pattern matching itself; it is a new feature of the switch expression, that accepts a type pattern as a case label.

In its current version, the switch expression accepts the following for the case labels:

1. the following numeric types: `byte`, `short`, `char`, and `int`
2. the corresponding wrapper types: `Byte`, `Short`, `Character` and `Integer`
3. the type `String`
4. enumerated types.

Pattern matching for switch expressions adds the possibility to use type patterns for the case labels.

### Future Directions for Pattern Matching

Pattern matching can be used in conjunction with deconstruction. Because the compiler knows that a record is built on its component, it can use this information to deconstruct a record and expose its internal state directly. This could make the following code possible in the future: 

```
record Rectangle(int width, int height) {}

// o is any variable
if (o instanceof Rectangle(int width, int height) {
   int surface = width*height;
}
```

Introducing deconstruction through factory methods could brings more possibilities. Suppose you have a map and need to extract the value bound to the key "name". You could write it in this way:

```
Map<String, String> map = ...; // any map
if (map instanceof Map.withMapping("name", String name)) {
   // you can use name in this block of code
}
```

## Wrap-up

In this exercise, you have used the **pattern matching for instanceof** feature, previewed in Java 14 and Java 15, and has been made a standard and permanent feature in Java 16. 

The **pattern matching for instanceof** feature unarguably simplifies the code but in this particular scenario, the '`if … else if …`' chain makes this code repetitive and potentially brittle! Wouldn't it be nice to use a `switch` instead of this '`if … else if …`' chain?  In fact, the **pattern matching for instanceof** feature along with the **Switch Expression** feature (see Lab 9), the traditional Switch statement, the **Records** feature (see Lab 7) and the **Sealed Class** feature (see Lab 8) will enable, in the near future, powerful pattern matching in the Java platform, including the ability to do pattern matching with Switch.

The **pattern matching for instanceof** feature supports one kind of pattern (type pattern) in one context (`instanceof`). This might seems limited but it is certainly more than a small nice-to-have improvement to simply save a few keystrokes! It is in fact one of the multiple new Java language features that together are slowly but surely paving the way for powerful pattern matching in the Java platform!

**Resources:**

* [Pattern Matching for instanceof JEP](https://openjdk.java.net/jeps/394)
* [Java Feature Spotlight: Pattern Matching](https://www.infoq.com/articles/java-pattern-matching/)
* [Pattern Matching in the Java Object Model](https://github.com/openjdk/amber-docs/blob/master/site/design-notes/pattern-match-object-model.md)


