# Sealed Classes

<div style="display: none;"><span><img src="https://billy.delabassee.com:8080/p/odl-16-lab/8"></span></div>

## Overview

In this 10-minutes lab, you will get some hands-on experiences with **Sealed Classes** (JEP 409), a feature of Java 17. This new feature enables the ability to create restricted class hierarchy. That is a class or interface that can declaratively restrict which other classes or interfaces may extend or implement them.


💡 Despite its name, the **Sealed Classes** feature applies to both **classes** and **interfaces**.

## Restricted  Class Hierarchies

In Java, a class hierarchy enables the reuse of code via inheritance: The methods of a superclass can be inherited (and thus reused) by many subclasses. However, the purpose of a class hierarchy is not always to reuse code. Sometimes, its purpose is to model the various possibilities that exist in a domain, such as the kinds of shapes supported by a graphics library or the kinds of loans supported by a financial application. When the class hierarchy is used in this way, restricting the set of subclasses can streamline the modeling.

A Sealed Class (or interface) can be extended (or implemented) only by those classes (and interfaces) explicitly permitted to do so.

* A new **`sealed` modifier** has been introduced to **seal** a class

* A new **`permits` clause** is then used to explicitly **specify** the class(es) that is(are) permitted to extend the sealed class 

Additionally a sealed class imposes three constraints on its permitted subclasses (the classes specified by its `permits` clause):

* The sealed class and its permitted subclasses must belong to the same module, and, if declared in an unnamed module, the same package.

* Every permitted subclass must directly extend the sealed class.

* Every permitted subclass must choose a modifier to describe how it continues the sealing initiated by its superclass:

    * A permitted subclass may be declared final to prevent its part of the class hierarchy from being extended further. It can also be implicitly final (enumerated types and records).
    * A permitted subclass may be declared sealed to allow its part of the hierarchy to be extended further than envisaged by its sealed superclass, but in a restricted fashion.
    * A permitted subclass may be declared non-sealed so that its part of the hierarchy reverts to being open for extension by unknown subclasses. (A sealed class cannot prevent its permitted subclasses from doing this.)
 

## Your first Sealed Classes

💡 **Sealed Classes** is a feature in JDK 17.

For the sake of this exercise, let us suppose that the conference application needs to deal with sessions of different types. 

Broadly speaking, the conference has the following sessions :

* **Breakout** session, each has a virtual room
	* **Lecture**: a traditional conference session
	* **Lab**: a hands-on lab
* **Keynote** session, a traditional general session

All extend the **Session** abstract class

1. Create a `session` directory (`mkdir -p src/main/java/conference/session/`) and create the abstract sealed `Session.java` superclass.

```nohighlight
<copy>
nano src/main/java/conference/session/Session.java
</copy>
```

```
<copy>
package conference.session;

import conference.Track;

import java.util.UUID;

sealed public abstract class Session
permits Keynote, Breakout {

    private String id;
    private String title;

    public Session(String id, String title) {
        this.title = title;
        //uid = UUID.randomUUID().toString();
        this.id= id;
    }

    public String getId() {
        return id;
    }

    public String getTitle() {
        return title;
    }
}
</copy>
```

🔎 `sealed public … class Session` ➡ declares it to be a **sealed** class.

🔎 `permits Keynote, Breakout …` ➡ explicitly declares that only the `Keynote` and the `Breakout` classes can extend it.


2. Now you need to create both `Keynote.java` and `Breakout.java` classes

```nohighlight
<copy>
nano src/main/java/conference/session/Keynote.java
</copy>
```


```
<copy>
package conference.session;

final public class Keynote extends Session {

    public String getKeynoteSpeaker() {
        return keynoteSpeaker;
    }

    String keynoteSpeaker;

    public Keynote(String id, String keynoteSpeaker, String title) {
        super(id, title);
        this.keynoteSpeaker = keynoteSpeaker;
    }
}
</copy>
```
🔎 `Keynote.java` is **final**, it can't be extended.


```nohighlight
<copy>
nano src/main/java/conference/session/Breakout.java
</copy>
```

```
<copy>
package conference.session;

import java.util.Random;

public sealed abstract class Breakout extends Session
permits Lab, Lecture {

    private String speaker;
    private int virtualRoom;

    public Breakout(String id, String title, String speaker) {
        super(id, title);
        this.speaker = speaker;
        this.virtualRoom = new Random().nextInt(3) + 1; // randomly assign a room
    }

    public String getSpeaker() {
        return speaker;
    }

    public int getVirtualRoom() {
        return virtualRoom;
    }
}
</copy>
```
🔎 `Breakout.java` is also **sealed**, it **permits** both the `Lab` and the `Lecture` classes to extend it.

3. Create the `Lecture.java` and `Lab.java` classes

```nohighlight
<copy>
nano src/main/java/conference/session/Lecture.java
</copy>
```

```
<copy>
package conference.session;

final public class Lecture extends Breakout {

    String slidesUrl;

    public Lecture(String id, String title, String speaker, String slidesUrl) {
        super(id, title, speaker);
        this.slidesUrl = slidesUrl;
    }

    public String getslidesUrl() {
        return slidesUrl;
    }
}
</copy>
```

```nohighlight
<copy>
nano src/main/java/conference/session/Lab.java
</copy>
```

```
<copy>
package conference.session;

final public class Lab extends Breakout {

    String labUrl;

    public Lab(String id, String title, String speaker, String labUrl) {
        super(id, title, speaker);
        this.labUrl = labUrl;
    }

    public String getLabUrl() {
        return labUrl;
    }
}
</copy>
```

🔎 Both classes are `final`.


4. Create a fictional `AgendaRepository.java` class

```nohighlight
<copy>
nano src/main/java/conference/AgendaRepository.java
</copy>
```

```
<copy>
package conference;

import conference.session.Keynote;
import conference.session.Lab;
import conference.session.Lecture;
import conference.session.Session;

import java.util.List;
import java.util.Optional;
import java.util.stream.Collectors;

public final class AgendaRepository {

    private final List<Session> sessionList;

    public AgendaRepository() {
        var keynote = new Keynote("001", "007", "The Future of Java Is Now");
        var s1 = new Lecture("005", "Java Language Futures - Early 2021 Edition", "021", "https://speakerdeck/s1");
        var s2 = new Lecture("006", "ZGC: The Next Generation Low-Latency Garbage Collector", "005", "https://slideshare/s2");
        var s3 = new Lecture("007", "Continuous Monitoring with JDK Flight Recorder (JFR)", "010", "https://speakerdeck/007");
        var hol1 = new Lab("010", "Building Java Cloud Native Applications with Micronaut and OCI", "030", "https://github.com/micronaut");
        var hol2 = new Lab("011", "Using OCI to Build a Java Application", "019", "https://github.com/011");

        sessionList = List.of(keynote, s1, s2, s3, hol1, hol2);
    }


    public List<Session> getAll() {
        List<Session> allSessions = sessionList.stream().toList();
        return allSessions;
    }


    public Optional<Session> getBySessionId(String sessionId) {
        Optional<Session> session = sessionList.stream()
                .filter(s -> s.getId().equals(sessionId))
                .findFirst();
        return session;
    }
}
</copy>
```

5. Create `AgendaService.java`

```nohighlight
<copy>
nano src/main/java/conference/AgendaService.java
</copy>
```

```
<copy>
package conference;

import conference.session.Keynote;
import conference.session.Lab;
import conference.session.Lecture;
import conference.session.Session;
import io.helidon.webserver.Routing;
import io.helidon.webserver.ServerRequest;
import io.helidon.webserver.ServerResponse;
import io.helidon.webserver.Service;

import java.util.List;
import java.util.logging.Logger;


public class AgendaService implements Service {

   private final AgendaRepository sessions;
   private static final Logger LOGGER = Logger.getLogger(AgendaService.class.getName());

   AgendaService() {
      sessions = new AgendaRepository();
   }

   @Override
   public void update(Routing.Rules rules) {
      rules.get("/", this::getAll);
   }

   private void getAll(final ServerRequest request, final ServerResponse response) {
     LOGGER.fine("getSessionsAll");

     List<Session> allSessions = this.sessions.getAll();
     response.send(allSessions);
   }
}
</copy>
```


6. Update the `createRouting` method in `Main.java` to instantiate the AgendaService and register its handler under the "/sessions" path.


```nohighlight
<copy>
nano src/main/java/conference/Main.java
</copy>
```

```
…
AgendaService sessionsService = new AgendaService();

…
return Routing.builder()
      …
      .register("/sessions", sessionsService)
      .build();

```
As you have probably guessed, you have just created an endpoint to exposes sessions details
It can be accessed via `{public_ip}:8080/sessions`.


7. Create a new session type.

The interesting part of the lab is the restricted `Session` classes hierarchy that you have created at the beginning. You can challenge it by creating, for example, a new session type (ex. `Quickie`) type that extends `Breakout`. Given that only `Lab` and `Lecture` are permitted to extend `Breakout`, the Java compiler will simply refuse that `Quickie` tries to extends `Breakout` but you should be able to fix this.

## Wrap-up

In this exercise, you have used **Sealed Classes**. Sealed Classes is a new feature that enables a developer to define a restricted classes hierarchy, i.e. a developer has now the ability to explicitly states for a given class (or an interface) which classes (or interfaces) may extend (or implement) it. Sealed Classes is a feature in JDK 17.

For more details, please check [JEP 409: Sealed Classes](https://openjdk.java.net/jeps/409) and the following [Java Feature Spotlight: Sealed Classes](https://www.infoq.com/articles/java-sealed-classes/) article.

You can also check the [JEP Café Episod #02](https://youtu.be/652kheEraHQ). 