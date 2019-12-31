# function === object with an `apply` method

## Goals

These examples illustrate how Scala functions are objects and objects (with `apply` methods) are functions.  These aren't all idiomatic usage and don't illustrate all the ways to express function objects.

They are offered with the following objectives:

1. To teach the main ways Scala "views" functions--as gerunds: nouns that are "verbed".
2. To introduce the concept of "function factories"--methods or functions that return new functions.

## The examples

Class instances can be functions

```scala
{
  class Square { def apply(x: Double) = x * x }
  val sqr = new Square
  sqr(5.0)
}
// res0: Double = 25.0
```

Writing functions as class instances is usually more useful when a constructor holds a parameter value constant.

```scala
{
  class Greeter(greeting: String) { def apply(who: String) = println(greeting + " " + who) }
  val greet = new Greeter("Howdy")
  greet("Slink")
  greet("Woody")
  greet("Buzz")
}
// Howdy Slink
// Howdy Woody
// Howdy Buzz
```

When there is no need for a parameter to be held constant (as in the first example), we can use a singleton object.

```scala
{
  object sqr { def apply(x: Double) = x * x }
  sqr(5.0)
}
// res2: Double = 25.0
```

Or you can use anonymous function syntax (which is a bit more concise) to define a function that is also an object.  (Int => Int is the function's type; it just says the function accepts an Int and returns an Int.)

```scala
{
  val sqr = (x: Int) => x * x
  println(sqr(5))
  println(sqr.apply(3))
}
// 25
// 9
```

A function object defined using anonymous function syntax also automatically extends Scala's built-in Function1[A,B] type, which provides other useful goodness that I'll let you look up.

```scala
{
  val sqr = (x: Int) => x * x
  sqr.isInstanceOf[Function1[Int,Int]]
}
// res4: Boolean = true
```

And using generics along with the Numeric implicits from Scala's standard library, we can define `sqr` over all Numeric types.

```scala
{
  import Numeric.Implicits._

  def sqr[T: Numeric](x: T) = x * x

  println(sqr(5))
  println(sqr(5.0))
}
// 25
// 25.0
```

Stay tuned!  There's much more fun to come!

## Let's apply what we've learned

(I suggest using [Ammonite](https://ammonite.io) or a Scala Notebook application to quickly try *New Scala Things* and immediately see your results.)

Here are a couple things to try based on what we learned above:

1. Create a function `double` with the type signature `Int => Int` (in the same way we wrote `val sqr` above) that returns twice its argument.  Similarly, write a function named `plusOne` that (obviously) adds one to its argument value.

2. We discussed one way to write function factories and we noted that functions created this way also have `Function1` as a supertype.  Let's play with this idea.  Two methods on `Function1` are `andThen` and `compose`.  Given `f1` and `f2` both of type T => T, these let us create a new function `f3` by writing `f1.compose(f2)` or `f1 compose f2`.  Using `double` and `plusOne` from above, play with `andThen` and `compose`.  What is their difference?  How might this technique be useful more generally?

