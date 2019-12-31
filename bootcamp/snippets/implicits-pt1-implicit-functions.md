### Prerequisites:

* [A class's companion](/bootcamp/snippets/classes-and-objects-pt2.md)
* [Objects can be functions](/bootcamp/snippets/objects-can-be-functions.md)
* [On Iterating without 'Looping'](/bootcamp/snippets/functions-and-loops.md)

--------

# The many faces of `implicit` modifiers -- implicit functions

Scala's `implicit` modifier can do different things in different contexts.  For example:

* An implicit function defines an automatic type conversion from some type to another.
* An implicit class defines a decorator pattern class--that is automatically applied to add new methods to existing classes.
* Implicit values are automatically passed to any implicit parameter list whose parameter type(s) match the values' types.
* Implicit values/parameter lists can be combined with generic type parameters to constrain the implicit value that can match an implicit parameter.  These generic type constraints combined with implicit values/parameters implement the typeclass pattern in Scala.

Let's look at each of these patterns in turn, beginning with implicit functions.

## Implicit functions

> **Why implicit functions?**
> 
> Implicit functions apply a type conversion automatically.

Here's a somewhat-frivolous example to start.

Suppose we write a function to concatenate words together into a sentence:

```scala
def toSentence(words: String*) = words.mkString("", " ", ".")

toSentence("Have", "a", "Coke", "and", "a", "smile")
// res0: String = "Have a Coke and a smile."
```

Suppose we want to include arbitrary integers in a sentence without explicitly calling a conversion function/method.  We could do it like this:

```scala
object Dictionary {
    import scala.io.Source
    val dict = Source.fromFile("/usr/share/dict/words").getLines.toArray

    def word = dict((Math.random * dict.size).toInt) // A random word, please
}

object sentence {
    import scala.language.implicitConversions
    implicit def i2s(i: Int): String = (i >> 16).asInstanceOf[Char].toString + (i & 0xffff).asInstanceOf[Char]

    def apply() = {
        import Dictionary._
        toSentence(word, word, word, "and", "a", 0xd83edd14)
    }
}

sentence()
// res1: String = "tinselling stranger pursuit and a \ud83e\udd14."
```

Internally, when Scala sees the Int being passed as a String parameter, before marking this as an error Scala looks for an implicit function that can convert an Int to a String.  If it finds one, it wraps the Int in a call to this function, thus resolving the compile error.  In other words, Scala is automatically rewriting the `toSentence` call inside the `apply()` method like this:

```scala
import Dictionary._  // Repeated from the object above  // Repeated from the object above
toSentence(word, word, word, "and", "a", sentence.i2s(0xd83edd13))
// res2: String = "drawers curries scuppered and a \ud83e\udd13."
```

The result is code that looks like it's written in a dynamic language but is still fully type checked.

## This looks really dangerous

Yes, it is!  There are two primary concerns here:

* Implicit conversion functions express very general and high-level meaning in a program; if that meaning isn't true universally throughout one's program, they can lead to unexpected and undesired results.  Following from this:
* Implicit conversion functions need to be valid in limited scopes.

Let's look closer.

### Implicit conversion functions are best used to express universal truths.

One probably never wants to create an automatic Int to String conversion for embedding unicode characters because different parts of our code might have different interpretations of int-to-String conversions.

For example, one part of our program might need the binary int value to be directly embedded inside a String and other parts are likely to need different conversion semantics (e.g.: a JSON library would more likely want to create a String containing the readable numeric value).

In practice, implicit conversion functions are often a convenient way to implement the Adapter Pattern--where two objects use different APIs to implement the same idea and need a third object to translate between them.  A trivial expression of this idea might be to adapt a Java API expecting a Runnable to a Scala function.

For example imagine we had the following API and want to call it from Scala (pretend this snippet is written in Java):

```scala
def doSomethingThen(callThis: Runnable): Unit = callThis.run()
```

Since Runnable is a Java construct, without implicit functions we would still need to call this the same way as Java:

```scala
doSomethingThen(new Runnable { override def run = println("This was called")})
// This was called
```

But using an implicit function, we can automatically convert between a Scala function `() => Unit` (or `Function0[Unit]`, which means the same thing) and a Runnable.  This lets us pass a regular Scala function directly to `doSomethingThen` **instead** of a Runnable and the Scala compiler will automatically write the adapter between Scala's function and Java's Runnable for us.  Here's one way to write the implicit conversion:

```scala
import scala.language.implicitConversions

implicit def function2Runnable(f: Function0[Unit]): Runnable = new Runnable {
  override def run = f()
}
```

Now we can call `doSomethingThen` and pass a regular Scala function, resulting in idiomatic Scala code even though we're dealing with a crufty old Java library.

```scala
doSomethingThen( () => println("This was called again"))
// This was called again
```

Here's another way to do this:

```scala
def doThis: Unit = println("again and again and...")

doSomethingThen(doThis _)  // the underscore passes "doThis" as a function
// again and again and...
```

Adapting idiomatic Scala to another JVM language's conventions is one legitimate use for implicit conversion functions.  Other legitimate uses for implicit functions typically address problems with a similar kind of scope.

### Implicit functions are only applied in the scope in which they're imported

We have described how to write implicit functions and their main use: solving very general problems requiring adapters or drivers.  But sometimes even when an implicit function solves a general problem we still need to limit the scope in which it can be applied.  Here's how:

> Implicit conversion functions are only applied when they have explicitly been imported.

Earlier, we defined the implicit conversion local to the `sentence` object.  That object, or things that implicitly `import sentence._` are the only places that implicit conversion will apply.

Isn't this still dangerous?

In this case, probably, since Int and String are used so widely.  The usual caveats for dynamic languages apply here.  At Rally, doing this will almost certainly get your PR *failed*.

Are there other places where implicit type conversions might make sense?

The only occasional use I've seen is between simple and complex types, not among simple types.  An example might be something like the following:

```scala
object FileImplicits {
    import scala.language.implicitConversions
    import java.io.File

    implicit def string2File(path: String) = new File(path)
    implicit def file2String(f: File): String = f.getCanonicalPath
}

import FileImplicits._  // and then pass Strings directly to functions requiring File objects
```

Even for cases like this, be prepared to defend why your implementation benefited from using implicit conversions and how your particular implementation isn't dangerous to the rest of the code base.

## Conclusion

Since implicit conversions are one of the less-used kinds of implicits, why did we cover it first?  Because they provide a foundation for implicit classes.  We'll cover those next.  😊

## Things to try

Implicit conversions can be used to mimic dynamic typing found in other languages.  Since it's dangerous to implement implicit conversions directly between built-in types (e.g.: from Int to Float), how might one achieve a similar effect using `trait`s and/or `case class`es as an intermediate form?

Implement dynamic type behavior for `Int`, `String`, and `Double` using one or more intermediate types.
