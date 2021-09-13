## How Records Address Serialization

When developers need to serialize and deserialize an object graph, in the vast majority of cases the goal is the transmission of the *data* contained within the object graph, not the programmatic behavior in the object graph. As you see the desire to try to reconstitute the entire object graph is the source of many of the woes with Java serialization.

Introduced in Java 16, [JEP 395](https://openjdk.java.net/jeps/395), Records address the outstanding issues with serialization by being transparent carriers of data. To achieve that goal, Records have several design constraints including:

* A Record's superclass is always `java.lang.Record`
* Records are implicitly `final` and cannot be `abstract`
* All fields in a record are `final` and derived from the components defined in its declaration
* Instance fields cannot be declared and cannot have instance initializers 
* Explicit declarations of a derived member must match the type of automatically derived member exactly
* Cannot contain `native` methods

These constraints allow Records to be serialized from its public accessors and deserialized using its canonical constructor. This also means that the serialization and deserialization of a Record class cannot be modified by implementing any of the "magic" methods: `writeObject`, `readObject`, `readObjectNoData`, `writeExternal`, or `readExternal`.

These constraints and behaviors of Records close the loop on many of the encapsulation concerns of serialization and deserialization. 