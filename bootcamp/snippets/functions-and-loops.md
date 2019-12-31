# On Iterating without "Looping"

Coming from OO languages, we're used to loops that start with a beginning state and build up an ending state, which terminates the loop.

In functional programming, we're not normally allowed to mutate state.  So how do we do this?  Here are a few examples to get you started.

## Loopy examples

Coming from a C-inspired language like Java or C#, one might expect loops to depend on a variable or a condition that changes under one or the other of the following circumstances:

1. Over time
2. When the loop needs to terminate

Here are two examples illustrating both of the above in order:

```java
public static class Loopy1 {
  public static String loopOverTime() {
    String[] onesies = new String[] {"one", "two", "three"};
    int i = 0;
    String intermediateResult = "";

    while (i <= onesies.length) {
      intermediateResult += " " + onesies[i];
      i++;                // Each loop iteration changes the index / termination condition
    }

    return intermediateResult;
  }
}
```

Or:

```java
public static class Loopy2 {
  public static String loopUntilTerminal(int atLeastlength, String toAppend) {
    String intermediateResult = "";
    boolean done = false;

    while (!done) {
      intermediateResult += toAppend;
      if (toAppend.getLength() >= atLeastlength) {
        done = true;      // We're terminal!  Finished!  Done!
      }
    }

    return intermediateResult;
  }
}
```

Notice how both of these loopy examples depend on destructively mutating state in order to build up a result and to know when the loops are done.

How could one eliminate the state destruction and only have state transformation instead?

## Transforming destruction into transformation

In Java, we might use recursion.  For the second example, we might write something like this:

```java
public static class Transformitive1 {

  private static buildString(int atLeastLength, String toAppend, String currentString) {
    if (currentString >= atLeastLength) {
      return currentString;
    } else {
      buildString(atLeastLength, toAppend, currentString + toAppend);
    }
  }

  public static String loopUntilTerminal(int atLeastLength, String toAppend) {
    return buildString(atLeastLength, toAppend, "");
  }
}
```

This addresses one loopy case; how might we address all of them?

In Java, the new `Streams` methods often help abstract these loops away.  On the other hand, Scala supplies a rich set of library methods that usually makes it possible to avoid writing explicit recursion like we did above--for all collections.  And Scala can make recursion a bit nicer to express too.

Here are a few examples:

### Transform all As into Bs

Given an `Int` sequence of As in the mathematical range `[1, 20]`, we provide a mapping function from any given `A` to its corresponding `B`.  Scala's `map` method loops for us.

```scala
{
  val As = 1 to 8
  val Bs = As.map { a: Int => a * a }
  Bs
}
// res0: collection.immutable.IndexedSeq[Int] = Vector(
//   1,
//   4,
//   9,
//   16,
//   25,
//   36,
//   49,
//   64
// )
```

### Accumulate results over time

Now let's rewrite Loopy1.  Here we start with a collection of Strings; we would like to reduce them to a single String--the concatenation of all strings in the collection.  So we'll use the `reduce` method:

```scala
{
  val As = Array("one", "two", "three")
  As.reduce { (intermediateResult: String, nextA: String) => intermediateResult + " " + nextA}
}
// res1: String = "one two three"
```

`reduce` initially passes element `0` and `1` to our reducer function as `(intermediateResult, nextA)`.  All subsequent times, `intermediateResult` is the result of concatenating `intermediateResult`, a space, and `nextA` from the prior time until As are gone.  Then, just like our Java implementation, `intermediateResult` is returned as the final result.

With practice, this is much nicer than writing the loop over and over again.  But we can do even better!

Since concatenating a sequence of Strings is such a common operation, Scala provides a special function, `mkString`, for just this situation:

```scala
{
  val As = Array("one", "two", "three")
  As.mkString(" ")
}
// res2: String = "one two three"
```

### Loop until a condition is true

Finally, let's rewrite Loopy2.  For this, an idiomatic Scala solution will most likely use the following principles:

* In Scala, everything returns a value (including `if` statements)
* Scala allows us to nest functions inside of functions as a way of limiting scope visibility
* Tail recursion, so the compiler can automatically rewrite the recursion as a loop

```scala
def loopUntilTerminal(atLeastLength: Int, toAppend: String) = {

  @scala.annotation.tailrec   // Mark this method as tail recursive
  def buildString(atLeastLength: Int, toAppend: String, currentString: String): String = {
    if (currentString.length >= atLeastLength) {
      currentString
    } else {
      buildString(atLeastLength, toAppend, currentString + toAppend)
    }
  }

  buildString(atLeastLength, toAppend, "")
}

loopUntilTerminal(5, "As")
// res3: String = "AsAsAs"
loopUntilTerminal(9, "x")
// res4: String = "xxxxxxxxx"
```

Like the Java recursive solution, Scala's allows us to avoid destructive state mutation, but without needing as much ceremony to express our solution.

## Conclusion

Pure functional programming prefers transforming state over destructively mutating it.  Scala's standard library offers a rich group of operations making it easier to work with collections and/or sequences of data without needing to destructively mutate state.  The examples above only scratch the surface; more information can be found browsing the Scaladoc under the Scala.collections hierarchy.

Because everything in Scala evaluates to a value (even `if` statements) and Scala's nesting rules allow us to restrict the scope of a function by nesting it inside another function, Scala makes it easier to construct recursive solutions that read well and that often are tail recursive--that Scala's compiler can automatically rewrite as a loop for efficiency's sake.

## Let's apply what we've learned

(I suggest using [Ammonite](https://ammonite.io) or a Scala Notebook application to quickly try *New Scala Things* and immediately see your results.)

Here are a couple things to try based on what we learned above:

* Implement a function `encode` implementing a substitution cipher using the following parameters:

  * Upper-case the input `String`
  * Increment the ASCII value of each character by 1
  * Return the result as a `String`

  Methods you will need include `toUpper`, `toChar`, `map`, and `mkstring`.

* Implement a `decode` function that is able to retrieve the plain text of a string encoded via `encode` (with limitations).

