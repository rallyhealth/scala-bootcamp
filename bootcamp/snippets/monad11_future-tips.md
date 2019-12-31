### Prerequisites:

* [Implicit values and parameter dependency injection](/bootcamp/snippets/implicits-pt4-implicit-values.md)
* [Introduction to Future[T]](/bootcamp/snippets/monad10_future-introduction.md)

--------

# `Future[T]` Tips and Techniques

Here we cover:

* A technique for unit testing `Future` code using a custom synchronous `ExecutionContext`
* How to obtain `Future[TestResults]` inside unit tests
* How to convert a `List[Future[T]]` to a `Future[List[T]]`
* What about a `Future[Future[T]]`?  (Also known as how to untie the knot one has made)
* How to recover when a `Future` calculation has failed

## What is an `ExecutionContext`?

Last time we promised to explain `ExecutionContext`; let's start there.

```scala
import scala.concurrent._
import scala.concurrent.ExecutionContext.Implicits.global
```

The above two imports are the usual imports required when using `Future`s.  The initial one imports `Future` itself; the second one does two additional things:

* It imports the global `ExecutionContext`.
* It makes this global `ExecutionContext` available as an `implicit val`.  (If you don't remember what an `implicit val` is, see the prerequisites listed above.)

From this we can already infer how an `ExecutionContext` is normally used: by importing one as an `implicit val`.  And Scala helpfully provides a global one we can simply import if we're happy to accept its default parameters.

But this doesn't yet explain what an `ExecutionContext` is.

### More detail, please

In short, an `ExecutionContext` provides a consistent API to the thread pool where a given `Future` will execute.  It may wrap and use a Java `Executor` but may be implemented other ways too.  For example, here's one way to do it:

```scala
import scala.concurrent.ExecutionContext

ExecutionContext.fromExecutor(new ThreadPoolExecutor( /* thread pool configuration */ ))
```

## A custom `ExecutionContext` for unit tests

Here's an `ExecutionContext` that doesn't use a thread pool at all but runs *synchronously*.  This illustrates:

* What's inside an `ExecutionContext`
* A technique that can sometimes be useful for eliminating timing variations when authoring unit tests
* A really bad idea for production code

```scala
import scala.concurrent.ExecutionContext

implicit val ec = new ExecutionContext {
  def execute(runnable: Runnable): Unit = runnable.run()
  def reportFailure(cause: Throwable): Unit = cause.printStackTrace()
}
// ec: AnyRef with ExecutionContext = repl.Session$App$$anon$1@44dbcccd
```

## How to obtain test results from a `Future[T]`

Unit tests involving `Future`s commonly need to exercise code run inside of an `ExecutionContext`, then extract the `Future`'s result in order to run assertions.  The way to do this is via the `Await` object.

```scala
import scala.concurrent._
import scala.concurrent.duration._

val someFutureCalculation = Future { "extremely time-consuming result" }
// someFutureCalculation: Future[String] = Future(Success(extremely time-consuming result))

val result = Await.result(someFutureCalculation, 1 nanos)
// result: String = "extremely time-consuming result"

assert(result.contains("time"))
```

## What to do with a `List`, `Seq`, or somesuch of `Future[T]`

Sometimes life just throws each of us a `List[Future[Thing]]` to deal with and it would be much more convenient to handle the entire `List` once in the `Future[List[Thing]]` instead of using lots of little future steps.  `Future.sequence` is our friend here.

```scala
// Let's say that our `Thing` is an `Int` for purposes of discussion, then...
type Thing = Int

def newFutureThing(thing: Thing) = Future(thing)

val futureThings = (1 to 3).toList.map(newFutureThing)
// futureThings: List[Future[Thing]] = List(
//   Future(Success(1)),
//   Future(Success(2)),
//   Future(Success(3))
// )

val dealWithItAllAtOnceInTheFuture = Future.sequence(futureThings)
// dealWithItAllAtOnceInTheFuture: Future[List[Thing]] = Future(Success(List(1, 2, 3)))
```

## How does one to flatten a `Future[Future[Thing]]` into just a `Future[Thing]`?

Using `flatten`.  Of course. ;)

```scala
val futureFutureThing: Future[Future[Thing]] = Future(newFutureThing(999))
// futureFutureThing: Future[Future[Thing]] = Future(Success(Future(Success(999))))

val justAFutureThing: Future[Thing] = futureFutureThing.flatten
// justAFutureThing: Future[Thing] = Future(Success(999))
```

## How to recover from a `Future(Failure())` by supplying a default value

Use your `Future`'s `recover` method.

```scala
import scala.util.control._

Future[String] {
  throw new IOException("Couldn't read String!")
  "A useless return value that will never be returned"
}.recover {
  case NonFatal(e) => "Use this default value instead"
}
// res1: Future[String] = Future(Success(Use this default value instead))
```

