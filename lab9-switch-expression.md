# Lab 9: Switch Expression

## Overview


This 10-minute lab will introduce you to the **Switch Expression** feature, a standard and permanent feature since Java 14. 

The **Switch Expression** augments the traditional Switch Statement to address some of its irregularities and annoyances including the well known 'fall through' problem, the support for multiple constants per case, the ability to enforce exhaustiveness, an improved scoping, etc.

## Switch Expressions in more details

The Switch Expression feature introduces a new form of switch label, "`case L ->…`" to signify that only the code to the right of the label is to be executed if the label is matched (without fall through!). It is now also possible to allow multiple constants per case (ex. "`case L, M, N ->…`"). And contrary to a Switch Statement, a Switch Expression can produce values. For example, the following Switch Expression returns the length of a day using an enum.

```
int numLetters = switch (day) {
    case MONDAY, FRIDAY, SUNDAY -> 6;
    case TUESDAY                -> 7;
    case THURSDAY, SATURDAY     -> 8;
    case WEDNESDAY              -> 9;
};
```
The Switch Expression also introduces a new `yield` statement to yield such value. And contrary to Switch Statement, exhaustiveness is enforced in Switch Expression, i.e. for all possible values, there must be a matching switch label. For additional details on Switch Expressions, please check [Switch Expressions (JEP 361)](https://openjdk.java.net/jeps/361).

💡 Given that the Switch Expression feature is a standard feature since Java 14, it is not necessary to use the `enable-preview` flags to use this feature. Do note that the Conference application still requires those flags as it uses some preview features.

## Add Switch Expressions

When browsing conference users, you can notice that the value used for the track is simply the value of the `Track.java` enumeration converted to a string, that is a bit crud (ex. it is in uppercase). 

![](images/lab9-1.png " ")


In this exercise, you will use a Switch Expression to produce more user-friendly track names. You can either continue to modify the previous exercise or start from a clean codebase by checking out the Lab9 branch: `git checkout lab9`

In the `SpeakerService.java` class, add the following `getTrackDetail` method:

```
<copy>
private String getTrackDetail(Speaker speaker) {
        
   String trackDetail;

   switch (speaker.track()) {
      case DB :
         trackDetail = "Oracle Database track";
         break;
      case JAVA :
         trackDetail = "Java track";
      case MYSQL :
         trackDetail = "MySQL track";
         break;
      default :
         trackDetail = "TBC";
         break;
   }
   return trackDetail;
}
</copy>
```

This snippet is straight-forward. It is a traditional switch statement that, based on an enumeration, will return a more user-friendly track name. Note that there is a default value.

Now update the `getSpeakersById` method as follow.

```
<copy>
private void getSpeakersById(ServerRequest request, ServerResponse response) {
   LOGGER.fine("getSpeakersById");

   String id = request.path().param("id").trim();

   record SpeakrDetails(String id, String name, String title, String company, String trackName) {}

   try {
      if (Util.isValidQueryStr(response, id)) {
         var match = this.speakers.getById(id);
         if (match.isPresent()) {
            var s = match.get();
            response.send(new SpeakrDetails(s.id(),
                  s.firstName() + " " + s.lastName(),
                  s.title(),
                  s.company(),
                  getTrackDetail(match.get()))
               );
        }
		else Util.sendError(response, 400, "getSpeakersById not found: " + id);
      }
   } catch (Exception e) {
      Util.sendError(response, 500, "Internal error! getSpeakersById: " + e.getMessage());
   }
}
</copy>
```

This method is simple. It first defines a new "SpeakrDetails" local record, and then if a speaker for a given ID is found, it creates a record for that speaker. Note that the new `getTrackDetail` is being used to populate the record track component. Finally, it simply sends the JSON representation of this speaker record back to the client.

💡 This method also leverages the local record feature introduced in Lab7.

When you now request speaker details via an ID (ex. `curl http://{public_ip}:8080/speakers/010`), you will get all details including a better track name.

Now there are multiple issues with this code!

First, if you look closely, you will notice an issue as all speakers of the Java track are now wrongly listed as speaking in the MySQL track. This bug is a consequence of a 'fall-through' switch. If you look at the switch statement, you will notice there's no `break` in the "JAVA" case.
```
…
case JAVA :
   trackDetail = "Java track";
case MYSQL :
   trackDetail = "MySQL track";
   break;
…
```

Given there is no `break`, the "JAVA" case will "fall through" to the next case until it reaches a `break`. That explains why we see the "MySQL" for all speakers from the Java track. You can easily solve this bug by adding a `break` for the "JAVA" case. 

The second issue is tied to the "default" case. Given that the `Track.java` enumeration has only 3 possibles values and that those 3 values are actually tested in the switch, there's no point in having a "default" case as it is unreachable. You can fix this by simply removing the "default" branch.

The last problem is tied to the fact that this code is quite repetitive and verbose (ex. we assign a different value to the `trackDetail` string in all the branches, etc.) and also quite error-prone (ex. you know what it's like to forget a `break`!).

You can solve this by replacing that Switch statement with the **Swith Expression** below.

```
<copy>
private String getTrackDetail(Speaker speaker) {

   var trackDetail = switch (speaker.track()) {
      case DB -> "Oracle Database";
      case JAVA -> "Java";
      case MYSQL -> "MySQL";
   };

   return trackDetail + " track";
}
</copy>
```

A few things to note:

* Exhaustiveness is enforced, i.e. all possible track values are tested. To confirm this, simply remove one of the cases, you will notice that the compiler will complain with the "The switch expression does not cover all possible input values" error. You can solve this by either making sure all the cases are covered or by introducing a "default" case. 

* Values produced by Switch Expression have a common type.

* There is no fall through in Switch Expression, and hence no `break`.

* This example uses the simplified `case L -> value_to_return;` notation. The new `yield` statement can also be used to yield a value (see example below).

```
private String getTrackDetail(Speaker speaker) {

   var trackDetail = switch (speaker.track()) {
      case DB -> "Oracle DataBase";
      case JAVA -> {
         var track = speaker.track().toString().toLowerCase();
         yield (track.substring(0, 1).toUpperCase() + track.substring(1));
      }
      case MYSQL -> "MySQL";
    };

   return trackDetail + " track";
}
```

A switch expression can, like a switch statement, also use a traditional switch block with "`case L:`" switch labels (implying fall through semantics). In this case, values are yielded using the new yield statement.

```
var trackDetail = switch (speaker.track()) {
   case DB : 
      yield "Oracle Database";
   case JAVA:
      yield "Java";
   case MYSQL:
      yield "MySQL";
};
```
## Wrap-up

In this exercise, you have used *Switch Expressions*, a standard feature since Java 14.

Switch expressions complement nicely the traditional Swith statement by enabling to easily write less error-prone Switch cases. Moreover, the produced code is also more readable! For additional details, please check [JEP 361: Switch Expressions (Standard)](https://openjdk.java.net/jeps/361).






