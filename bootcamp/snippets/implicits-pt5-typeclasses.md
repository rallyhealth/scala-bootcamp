### Prerequisites:

* [Implicit values and parameter dependency injection](/bootcamp/snippets/implicits-pt4-implicit-values.md)
* [Objects can be functions](/bootcamp/snippets/objects-can-be-functions.md)
* [Traits are like interfaces](/bootcamp/snippets/classes-and-objects-pt3.md)

--------

# Introduction to Typeclasses

> *An **InputStream**;*  
> *A **randomThing**.*
> 
> *Singing?  Concurring?*

Previously we talked about how implicit values are a simple form of parameter dependency injection.  Typeclasses are a very specific way to use this feature.  Today we'll explore them.

## What is a typeclass?

A typeclass is an "adapter" between one thing and another, implicitly injected into a method.

What???

Let's break that down.

### An adapter

While typeclasses don't specifically require the Adapter Pattern, it's an easy place to start, so...

With the adapter pattern, an adapter `trait` (or `abstract class`) implements a common API used by a method/function to "adapt" one object's behavior or data to be used by another.  Or, said a bit differently, an adapter provides a "bridge" that allows two classes to work together.

### Between one thing and another

Here's an adapter for opening InputStreams from Random Things:

```scala
import java.io._
import java.net._
import scala.io.{Source, Codec}

trait StreamAdapter[SRT] { def apply(someRandomThing: SRT): InputStream }
```

Now with StreamAdapter[SRT] we can interface between SRTs (Some Random Things) and InputStreams like this:

```scala
def readRandomThingFromInputStream_ReturnItAsAListOfString[SRT](
  someRandomThing: SRT,
  openStream: StreamAdapter[SRT]
): List[String] = {
  val stream = openStream(someRandomThing)
  val content = Source.fromInputStream(stream).getLines.toList
  stream.close()
  content
}
```

And to use this, We'll need some StreamAdapter implementations:

```scala
class FileStreamAdapter extends StreamAdapter[File] { override def apply(f: File) = new FileInputStream(f) }
class UrlStreamAdapter extends StreamAdapter[URL] { override def apply(u: URL) = u.openStream }
```

Then we can `readRandomThingFromInputStream` using the adapter classes above.

```scala
readRandomThingFromInputStream_ReturnItAsAListOfString(new File("/etc/passwd"), new FileStreamAdapter).take(2)
// res0: List[String] = List(
//   "root:x:0:0:root:/root:/bin/bash",
//   "daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin"
// )
readRandomThingFromInputStream_ReturnItAsAListOfString(new URL("http://www.cs.columbia.edu"), new UrlStreamAdapter).take(2)
// res1: List[String] = List(
//   "<!DOCTYPE html>",
//   "<html lang=\"en-US\" class=\"no-js\">"
// )
```

This works, and is about the best we could do if this were Java.

But since we're writing in Scala, what if we used what we learned [last time](implicits-pt4-implicit-values.md) along with some clever type inference?

Our goal would be to automatically inject the correct StreamAdapter into an implicit parameter of our `readRandomThing...` method and we wouldn't have to explicitly pass any `StreamAdapter` implementation at all!  Here's how:

### Implicitly injected into a method

The idea is to use type inference to automatically pass the correct `implict val` as the adapter.  Here's what that looks like:

```scala
def readThing[SRT](someRandomThing: SRT)(implicit aStreamFrom: StreamAdapter[SRT]): List[String] = {
  val stream = aStreamFrom(someRandomThing)
  val content = Source.fromInputStream(stream).getLines.toList
  stream.close()
  content
}
```

In the method above, the type of `SRT` is defined by the type of the parameter passed as `someRandomThing`.  Then the StreamAdapter[SRT], by inference, also has to be type-compatible with `SRT`.

So, if we pass a `File` as `someRandomThing`, the `aStreamFrom` parameter must be a `StreamAdapter[File]`.  And since the `aStreamFrom` parameter is declared implicitly, we can inject our `StreamAdapter[File]` using an implicit val.

Like this:

```scala
implicit val fileAdapter = new FileStreamAdapter
implicit val urlAdapter = new UrlStreamAdapter
```

And when we call `readThing`, Scala will automatically infer the correct implicit val based on the type of the thing we pass as `someRandomThing`.

```scala
readThing(new File("/etc/passwd")).take(2)
// res2: List[String] = List(
//   "root:x:0:0:root:/root:/bin/bash",
//   "daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin"
// )
readThing(new URL("http://www.cs.columbia.edu")).take(2)
// res3: List[String] = List(
//   "<!DOCTYPE html>",
//   "<html lang=\"en-US\" class=\"no-js\">"
// )
```

And this is an example of a typeclass in action.

## Let's put it all together

Here's the entire typeclass example again, reworked to tidy a few things.

```scala
object TypeclassExample {
  import java.io._
  import java.net._
  import scala.io.{Source, Codec}

  // The trait is our typeclass
  trait StreamAdapter[SRT] { def apply(s: SRT): InputStream }

  // Typeclass instances
  implicit object FileStream extends StreamAdapter[File] { override def apply(f: File) = new FileInputStream(f) }
  implicit object UrlStream extends StreamAdapter[URL] { override def apply(u: URL) = u.openStream }

  // Typeclass usage: parameter injection along with type inference
  def thingAsStrings[SRT](someRandomThing: SRT)(implicit aStreamFrom: StreamAdapter[SRT]): List[String] = {
    val stream = aStreamFrom(someRandomThing)
    val content = Source.fromInputStream(stream).getLines.toList
    stream.close()
    content
  }
}
```

Here we're using `SRT` both as the type of `someRandomThing` and as the *type parameter* of the `aStreamFrom` implicit parameter.  The consequence is that the type of the object we pass as `someRandomThing` must match the type parameter of the `StreamAdapter` implementation that will be implicitly injected into `aStreamFrom`.

* If we pass a `File` to the `someRandomThing` parameter, the only thing that can be implicitly injected into the `aStreamFrom` parameter is an instance of `StreamAdapter[File]`: in the example above, the `FileStream` object.
* If we pass a `Url` to the `someRandomThing` parameter, the only thing that can be implicitly injected into the `aStreamFrom` parameter is an instance of `StreamAdapter[Url]`: in the example above, the `UrlStream` object.

In other words, in the following example:

```scala
import TypeclassExample._

thingAsStrings(new File("/etc/passwd")).take(2)
// res4: List[String] = List(
//   "root:x:0:0:root:/root:/bin/bash",
//   "daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin"
// )
thingAsStrings(new URL("http://www.cs.columbia.edu")).take(2)
// res5: List[String] = List(
//   "<!DOCTYPE html>",
//   "<html lang=\"en-US\" class=\"no-js\">"
// )
```

* Scala automatically chooses `FileStream` to retrieve content in the first case because we passed a `File` as the someRandomThing (`SRT`) parameter.
* Similarly, scala automatically chooses `UrlStream` to retrieve content in the first case because we passed a `Url` as the someRandomThing (`SRT`) parameter.

### So what?

In order to enable our `thingAsStrings` function to handle another data type, all we have to do is implement a new `StreamAdapter` implementation for our new data type and declare an implicit value for Scala to inject into the `aStreamFrom` parameter.

## Conclusion

What can we do with this?

* Using typeclasses, `thingsAsStrings` can handle any input type now or in the future, regardless of the inheritence hierarchies available or any other constraint currently present in our code.
* By decoupling the implementation of `thingsAsStrings` from its input type, we can trivially implement a `StreamAdapter` that injects test data.  The effect is to remove I/O operations from `thingsAsStrings` for testing purposes without resorting to heavy-weight alternatives like mocking frameworks.

This is the tip of the iceberg.  As you continue your Scala exploration you'll encounter many more uses for Typeclasses.
