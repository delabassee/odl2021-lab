## Random (Number) Generators

## Overview

In this 10-minute lab, you will explore the updated **Random Generator API**, an improvement of Java's existing API for generating randomness.
While leaving the specific APIs (with methods like `nextInt()` and `nextGaussian()`) mostly untouched, it introduces an inheritance hierarchy for classes that deal with random events and a uniform approach to instantiating them that is robust in the face of adding new and removing outdated algorithms.
This improvement was made in Java 17.

## `RandomGenerator` and other new interfaces

One core element of [the new API][random-api] is the introduction of a new interfaces hierarchy für random (number) generators.
At the top is [the new interface `RandomGenerator`][random-generator] with an API that's basically that of the preexisting class `Random` plus a few methods it was missing - for example those where you pass an upper bound for the next number.
Its implementations are not required to be thread-safe or cryptographically secure (i.e. they _can_ have one or both of those properties, but they don't _have_ to).

Then there are five additional interfaces that extend `RandomGenerator`.
Their differentiating factor is how you can use one such generator to create another one that is statistically independent and individually uniform (or some approximation thereof), so you can pass them off to a new thread.

`StreamableGenerator` can return a stream of random generators.
This is helpful because if a generator can create a set of new generators all at once, it can make a special effort to ensure that they're statistically independent.
How can a generator create another generator?
That depends on the underlying algorithm.
Some can return a new generator by jumping forward a number of draws - depending on who many, they implement `JumpableGenerator`, `LeapableGenerator`, or `ArbitrarilyJumpableGenerator`.

Then there's `SplittableGenerator`, which prescribes the behavior that the preexisting `SplittableRandom` already implements:
To use randomness across different threads, it has a method `split` that returns a new `SplittableRandom` that you can pass to a task in a newly spawned thread - this works great for fork/join-style computations.

All four existing classes `Random`, `SecureRandom`, `ThreadLocalRandom`, and `SplittableRandom` were expanded to provide the full API of `RandomGenerator`, which they now implement.
Furthermore, `SplittableRandom` also implements `SplittableGenerator`.
That means you can either keep using the existing classes or migrate towards the new interface to make it easier to exchange implementations.
Similarly to `List` vs `ArrayList` and `LinkedList`.

No new public classes were added, though, so how do you get an instance of, say, `LeapableGenerator`?

<img src="./images/random-generator-hierarchy.png" alt="Type hierarchy of the new Random Generator API as described in the text" width=100%/>

[random-api]: https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/random/package-summary.html
[random-generator]: https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/random/RandomGenerator.html

## Selecting algorithms by name

A few new algorithms have been implemented as internal aclasses and more will likely come in the future.
This is a complicated and evolving field, though, and so we're not going to explore any specific algorithm here.
Instead, we'll learn how to find an algorithm that fulfills a given set of requirements.

Generally speaking, all the new interfaces have a static `of` method that takes an algorithm name as a `String` argument.
If an algorithm of that name is implemented and adheres to the interface `of` is called on, it will return an instance of it.
Otherwise it throws an `IllegalArgumentException`.

```java
String algorithmName = "...";

RandomGenerator.of(algorithmName);
StreamableGenerator.of(algorithmName);
JumpableGenerator.of(algorithmName);
LeapableGenerator.of(algorithmName);
ArbitrarilyJumpableGenerator.of(algorithmName);
SplittableGenerator.of(algorithmName);
```

To experiment with this, create a new file with `nano RandomApi.java` and add the following code:

```java
import java.util.random.RandomGenerator;
import java.util.random.RandomGenerator.ArbitrarilyJumpableGenerator;
import java.util.random.RandomGenerator.JumpableGenerator;
import java.util.random.RandomGenerator.LeapableGenerator;
import java.util.random.RandomGenerator.SplittableGenerator;
import java.util.random.RandomGenerator.StreamableGenerator;
import java.util.random.RandomGeneratorFactory;

import static java.util.function.Predicate.not;

public class RandomApi {

	public static void main(String[] args) {
		String algorithmName = "Xoshiro256PlusPlus";

		RandomGenerator.of(algorithmName);
		StreamableGenerator.of(algorithmName);
		JumpableGenerator.of(algorithmName);
		LeapableGenerator.of(algorithmName);
		ArbitrarilyJumpableGenerator.of(algorithmName);
		SplittableGenerator.of(algorithmName);
	}

}
```

You can execute it with `java RandomApi.java`.
The result will be an `IllegalArgumentException` on the line `ArbitrarilyJumpableGenerator.of(algorithmName);` because _Xoshiro256PlusPlus_ does not implement that interface.
If you comment that line out, you get a very similar one on the next line because it's no `SplittableGenerator` either.
Comment out that line as well and the program works.

The Javadoc contains [a list of algorithms][algorithms] that all JDK implementations must contain, but any specific JDK may add more.
Due to advances in random number generator algorithm development and analysis, this list isn't set in stone, though:
Algorithms can be deprecated at any time and can be removed in major JDK versions.
So picking algorithms by name isn't the best way to write robust code.

[algorithms]: https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/random/package-summary.html#algorithms

## Selecting algorithms by properties

The second core element of the new API is the robust selection of algorithms based on requirements.
If you don't have any specific requirements, you can call `RandomGenerator.getDefault()` to get an arbitrary algorithm:

```java
RandomGenerator generator = RandomGenerator.getDefault();
```

If you need more control, you can use [the new class `RandomGeneratorFactory`][random-factory].
It also has a static `of(name)` method, which returns a factory for a specific algorithm, but that gets you back to where you started and doesn't help with writing more robust code.
For that you can call the static method `all`, which will return a stream of factories, one per algorithm.

The helpful side of that is that you can query factories for properties of the algorithm.
Is it jumpable, leapable, or splittable?
But also how many state bits it has, whether it uses a hardware device, or what its equidistribution is.
You can then filter by your specific requirements, find any factory that fulfills them, and use it to create a `RandomGenerator`.

To experiment with that, change `RandomApi.java` to the following:

```java
import java.util.random.RandomGenerator;
import java.util.random.RandomGenerator.JumpableGenerator;
import java.util.random.RandomGeneratorFactory;

import static java.util.function.Predicate.not;

public class RandomApi {

	public static void main(String[] args) {
		RandomGenerator generator = RandomGeneratorFactory.all()
			.filter(factory -> factory.stateBits() > 128)
			.findAny()
			.map(RandomGeneratorFactory::create)
			.orElseThrow();

		JumpableGenerator jumpableGenerator = RandomGeneratorFactory.all()
			.filter(RandomGeneratorFactory::isJumpable)
			.findAny()
			.map(RandomGeneratorFactory::create)
			.map(JumpableGenerator.class::cast)
			.orElseThrow();
	}

}
```

This executes successfully and shows how to pick a `RandomGenerator` with a specific property (in this case with more than 128 state bits) and a `JumpableGenerator` - both without naming any specific algorithm.

[random-factory]: https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/util/random/RandomGeneratorFactory.html

## Wrap-up

So, in summary:

* a new hierarchy of interfaces makes it easier to identify key properties of an algorithm and to switch between implementations
* the four existing random number classes got refactored, slightly extended, and now implement those interfaces while their behavior stays as is
* new algorithms have been and will be implemented as internal classes
* factories can be used to robustly pick algorithms that fulfill specific requirements
