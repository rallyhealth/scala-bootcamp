### Prerequisites:

* [Functions can return functions](/bootcamp/snippets/functions-returning-functions.md)
* [Monads for those who understand #map and #flatMap](/bootcamp/snippets/monad01_from-collections.md)
* [On Iterating without 'Looping'](/bootcamp/snippets/functions-and-loops.md)

--------

# What is a Future?

## Preface

Here are the imports required by the following code examples:

```scala
import scala.concurrent._
import scala.concurrent.ExecutionContext.Implicits.global

import scala.util.{Success,Failure}
import scala.io.Source
import java.io._
```

**Note:** The next article in this series will explain how `ExecutionContext` relates to `Futures` and `Executor`s.  Here we'll focus on what a `Future` is and how to use one.

## So you want to know the future

A `Future[T]` is a container for a value that will be computed in a background thread while the current thread continues executing.  Naturally, all caveats regarding how concurrent operations may complete out of order apply here.

```scala
{
  def futurePrint(x: Int): Future[Unit] = Future {
    Thread.sleep(300); println(s"f($x)")
  }

  val fs = (1 to 7).map(futurePrint)

  while (fs.exists( _.isCompleted == false)) {}  // Wait for all the futures above to complete
}
// f(2)
// f(3)
// f(1)
// f(4)
// f(7)
// f(6)
// f(5)
```

## Let's pretend to know the future...

A future can be constructed using the `Future { // some long calculation here }` syntax.

```scala
val futurePasswd = Future {
  Thread.sleep(500) // Pretend this will take a long time
  Source.fromFile("/etc/passwd").getLines.take(5).toList
}
```

We can obtain the value contained in the `Future` using its `value` method.

```scala
futurePasswd.value
// res2: Option[util.Try[List[String]]] = None
```

Let's examine the signature of the return type here, starting from outer to innermost type.

* **`Option`** - Some(value) if the Future has completed; None otherwise.
* **`Try`** - Success(result) if the Future completed successfully; Failure(exception) otherwise.
* **`List[String]`** - The actual result we want to compute.

These types illustrate the various states a Future may be in.

And since the `value` method cannot know if a `Future` has completed, it's also nondeterministic if the `Option` will be `Some[value]` or `None`.  This illustrates why it's almost never useful to call `.value` on a `Future`.  The value simply may not have been computed yet, as in the example above.

Since `.value` isn't very useful, what *are* some useful ways to obtain a Future's result?

## A Future is a container

We mentioned earlier that `Future`s are a kind of container.  Might this be useful?

It turns out that the answer is "yes" and that treating a `Future` as a kind of container is one of the more common ways to get values out of them.  Let's look at some of the ways:

### foreach

This is only rarely used--mostly for when the Future's result is used purely for side-effects.

```scala
futurePasswd.foreach( (l: List[String]) => l.foreach(println) )

Thread.sleep(750)  // Wait for futurePasswd to finish and print its results
// root:x:0:0:root:/root:/bin/bash
// daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
// bin:x:2:2:bin:/bin:/usr/sbin/nologin
// sys:x:3:3:sys:/dev:/usr/sbin/nologin
// sync:x:4:65534:sync:/bin:/bin/sync
```

### flatMap / map

`flatMap` and `map` can be useful when the results of multiple `Future`s need to be combined into a single result.

```scala
// Start two "long-running" operations in the background
val getProduct1234Info = Future { (1234, "Product description") }
// getProduct1234Info: Future[(Int, String)] = Future(Success((1234,Product description)))

val getProduct1234Picture = Future {
  val b = new ByteArrayOutputStream
  val p = new PrintStream(b)
  p.print("a beautiful image")
  b.toByteArray
}
// getProduct1234Picture: Future[Array[Byte]] = Future(Success([B@3caa3ba0))

// Combine both futures' results
getProduct1234Info.flatMap { case (productId, description) =>
  getProduct1234Picture.map { picture =>
    (productId, description, picture)
  }
}
// res5: Future[(Int, String, Array[Byte])] = Future(Success((1234,Product description,[B@3caa3ba0)))
```

There's a problem with this approach though; the nested `flatMap` and `map` operations can get out of control, making code hard to read.  Fortunately, there's a solution:

### for comprehensions

`for` comprehensions are actually [syntactic sugar for nested `flatMap`s, ending with a single `map`](../blog/2018-12-03-comprehending-for-comprehensions.md) and are usually a lot easier to read.  Here's the same example again, but rewritten as a `for` comprehension.

```scala
// Combine both futures' results
val getInfoToDisplay = for {
  productInfo <- getProduct1234Info
  picture <- getProduct1234Picture
} yield {
  val (productId, description) = productInfo
  (productId, description, picture)
}
// getInfoToDisplay: Future[(Int, String, Array[Byte])] = Future(Success((1234,Product description,[B@3caa3ba0)))
```

***Note:** The `<-` operator above is usally read "in".*

## Other ways of processing a Future's results

The above techniques are perfect when all `Future`s succeed, but they don't offer any mechanism for handling failure.  How might one address this?

Scala has another kind of callback specifically for this case.

```scala
getInfoToDisplay.onComplete {
  case Success(info) => println(info)
  case Failure(exception) => exception.printStackTrace()
}

Thread.sleep(500)  // Give the println time to complete
// (1234,Product description,[B@3caa3ba0)
```

## Conclusions

We've now explored the main ways to execute code in the background and process the results and/or errors when they occur.  But we haven't explored:

* How `Future`s relate to Java `Executor`s.
* What to do when you have a `Future` inside another `Future` or a `List[Future[T]]`.
* Promises
* ...

## Exercises

These are in increasing level of difficulty and we don't expect every engineer to get all of these answers.  If anything here remains mystifying after playing with these code examples, please ask in the "Backend Bootcamp" Hipchat room.

* As illustrated above, the main way Scala programs process `Future` results is to process those results in a callback and allow the current thread to process to completion.  Why would that be?
* Change `getInfoToDisplay` so that `getProduct1234Info` throws an exception and examine what happens.  Does `getProduct1234Picture` still execute?  Why or why not?
* Try the same with `getProduct1234Picture` and see what happens.  Can you explain the program's behavior?
* Here's a Scala newbie mistake with `Future`s; what's not great about rewriting the example above this way?

```scala
val getInfoToDisplay = for {
  productInfo <- Future {
    (1234, "Product description")
  }
  picture <-  Future {
    val b = new ByteArrayOutputStream
    val p = new PrintStream(b)
    p.print("a beautiful image")
    b.toByteArray
  }
} yield {
  val (productId, description) = productInfo
  (productId, description, picture)
}
```

