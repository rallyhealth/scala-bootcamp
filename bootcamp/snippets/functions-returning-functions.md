### Prerequisites:

* [Objects can be functions](/bootcamp/snippets/objects-can-be-functions.md)
* [On Iterating without 'Looping'](/bootcamp/snippets/functions-and-loops.md)

--------

# Functions can return functions

Here's an example illustrating one way functions/methods can return functions.

```scala
def bindParametersToFunction(p1: Int, p2: Int, p3: Int) : () => Int = {
  val result = () => p1 * p2 * p3
  result
}
```

This factory defines some calculation that needs to be executed later, but with parameters bound to values right now.  Here are a few examples of when this technique can be useful:

* When we need to sequence a number of calculations together, but we don't know the order in advance.
* To dynamically build a Map of named operations; an operation lookup table, if you will.
* To define one or more expensive operations to be executed in sequence inside a single Future.

Let's look at some code illustrating this final case.

```scala
import scala.concurrent.Future
import scala.concurrent.ExecutionContext.Implicits.global

val fns = List(
    bindParametersToFunction(1, 2, 3),
    bindParametersToFunction(2, 3, 4),
    bindParametersToFunction(3, 4, 5),
    bindParametersToFunction(22, 32, 42))
// fns: List[() => Int] = List(
//   <function0>,
//   <function0>,
//   <function0>,
//   <function0>
// )

val f = Future { fns.map( _.apply() ).reduce( _ + _ ) }
// f: Future[Int] = Future(Success(29658))
f.value
// res0: Option[util.Try[Int]] = None
fns.map( _.apply() ).reduce( _ + _ )
// res1: Int = 29658
```

## There's a name for this pattern

Here we've see an example of a function that returns another function as its result value.  This is one example of what functional programming practicioners call a "higher-order function".  So what's a higher-order function?

* Like we've seen here: a function that returns a function as its result.
* A function that accepts one or more functions as a parameter.  In Java or C#, observers are a common example of this.  In Scala, a lot of other functions do this, particularly those that function a data structure from M[A] to M[B] by iterating over M[A]'s contents and doing something to each element.

## Things to try

Object constructors are also a way to bind parameters to functions for future use.  For example:

```scala
class Greeter(greeting: String) {
  def greet(directObject: String) = s"$greeting $directObject"
}

val formal = new Greeter("Hello")
// formal: Greeter = repl.Session$App$Greeter@5085dc6b
formal.greet("Frank")
// res2: String = "Hello Frank"
```

Another way to implement constructor behavior uses a function that returns a function--a higher-order function.

```scala
def greeter(greeting: String) = { directObject: String => s"$greeting $directObject" }

val cowboyGreet = greeter("Howdy")
// cowboyGreet: String => String = <function1>
cowboyGreet("Frank")
// res3: String = "Howdy Frank"
```

Experiment with functions returning functions as a substitute for domain objects.  When would you want to use this technique?  When not?
