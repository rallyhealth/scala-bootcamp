### Prerequisites:

* [Implicit classes and open classes](/bootcamp/snippets/implicits-pt2-implicit-classes.md)
* [Objects can be functions](/bootcamp/snippets/objects-can-be-functions.md)
* [On Iterating without 'Looping'](/bootcamp/snippets/functions-and-loops.md)

--------

# Case study: `implicit` classes can help fix API incompatibilities

In our [previous installment](implicits-pt2-implicit-classes.md), we introduced implicit classes as a way to add new methods to existing classes.  Here we'll offer a practical application.

Given any random function object, we can invoke it as a standalone function:

```scala
val plus = (a: Int, b: Int) => a + b
// plus: (Int, Int) => Int = <function2>
plus(3, 2)
// res0: Int = 5
```

Suppose that in addition to this we want to invoke it using data already stored in a Tuple[T]?  Scala provides a built-in way to do this too.

```scala
val args = (3, 2)
// args: (Int, Int) = (3, 2)
plus.tupled(args)
// res1: Int = 5
```

By itself this is rarely interesting; having an argument list conveniently already stored inside a Tuple does happen, but not very often.

More frequently a function's arguments may be stored in a `List`, `Seq`, or other ordered collection.  But Scala does not supply a built-in way to convert from a Seq-like thing into a Tuple with the same number of elements.

We can simulate this by hand using a `case` construct:

```scala
val sesameStreet = Array(
    plus.tupled( Seq(1, 1) match { case Seq(a, b) => (a, b)} ),
    plus.tupled( List(2, 2) match { case Seq(a, b) => (a, b)} ),
    plus.tupled( Stream(4, 4) match { case Seq(a, b) => (a, b)} ),
    plus.tupled( scala.collection.mutable.ListBuffer(8, 8) match { case Seq(a, b) => (a, b)} ),
    plus.tupled( Vector(16, 16) match { case Seq(a, b) => (a, b)} ))
// sesameStreet: Array[Int] = Array(2, 4, 8, 16, 32)
```

Here, the very same `case` construct is used to pull apart any kind of sequential collection and put its contents into a tuple with the same arity as the container's `size`.

This seems serviceable, but to my eyes all the extra line noise obscures this code's intent.  Wouldn't it be nice if all `Seq`-like things had a method allowing us to convert a `Seq` to a Tuple with the same arity as the number of elements in the `Seq`?

Using implicit classes like we've introduced previously we can do it.  🏵️

## Sequish things to Tuppleish things

Let's add a generic `.toTuple` method to `Seq` and all of its subclasses:

```scala
implicit class Tuplizer[T](s: Seq[T]) {
    def toTuple[TUP <: Product]: TUP = {
        s match {
            // Tuple0 .. Tuple22, the largest one
            case Seq(a) => Tuple1[T](a)
            case Seq(a,b) => (a,b)
            case Seq(a,b,c) => (a,b,c)
            case Seq(a,b,c,d) => (a,b,c,d)
            case Seq(a,b,c,d,e) => (a,b,c,d,e)
            case Seq(a,b,c,d,e,f) => (a,b,c,d,e,f)
            case Seq(a,b,c,d,e,f,g) => (a,b,c,d,e,f,g)
            case Seq(a,b,c,d,e,f,g,h) => (a,b,c,d,e,f,g,h)
            case Seq(a,b,c,d,e,f,g,h,i) => (a,b,c,d,e,f,g,h,i)
            case Seq(a,b,c,d,e,f,g,h,i,j) => (a,b,c,d,e,f,g,h,i,j)
            case Seq(a,b,c,d,e,f,g,h,i,j,k) => (a,b,c,d,e,f,g,h,i,j,k)
            case Seq(a,b,c,d,e,f,g,h,i,j,k,l) => (a,b,c,d,e,f,g,h,i,j,k,l)
            case Seq(a,b,c,d,e,f,g,h,i,j,k,l,m) => (a,b,c,d,e,f,g,h,i,j,k,l,m)
            case Seq(a,b,c,d,e,f,g,h,i,j,k,l,m,n) => (a,b,c,d,e,f,g,h,i,j,k,l,m,n)
            case Seq(a,b,c,d,e,f,g,h,i,j,k,l,m,n,o) => (a,b,c,d,e,f,g,h,i,j,k,l,m,n,o)
            case Seq(a,b,c,d,e,f,g,h,i,j,k,l,m,n,o,p) => (a,b,c,d,e,f,g,h,i,j,k,l,m,n,o,p)
            case Seq(a,b,c,d,e,f,g,h,i,j,k,l,m,n,o,p,q) => (a,b,c,d,e,f,g,h,i,j,k,l,m,n,o,p,q)
            case Seq(a,b,c,d,e,f,g,h,i,j,k,l,m,n,o,p,q,r) => (a,b,c,d,e,f,g,h,i,j,k,l,m,n,o,p,q,r)
            case Seq(a,b,c,d,e,f,g,h,i,j,k,l,m,n,o,p,q,r,s) => (a,b,c,d,e,f,g,h,i,j,k,l,m,n,o,p,q,r,s)
            case Seq(a,b,c,d,e,f,g,h,i,j,k,l,m,n,o,p,q,r,s,t) => (a,b,c,d,e,f,g,h,i,j,k,l,m,n,o,p,q,r,s,t)
            case Seq(a,b,c,d,e,f,g,h,i,j,k,l,m,n,o,p,q,r,s,t,u) => (a,b,c,d,e,f,g,h,i,j,k,l,m,n,o,p,q,r,s,t,u)
            case Seq(a,b,c,d,e,f,g,h,i,j,k,l,m,n,o,p,q,r,s,t,u,v) => (a,b,c,d,e,f,g,h,i,j,k,l,m,n,o,p,q,r,s,t,u,v)
            case _ => throw new IllegalStateException("Can't build a Tuple with more than 22 elements")
        }
    }.asInstanceOf[TUP]  // Downcast to the expected return type: Dangerous!
}
```

With this in place, our gnarly `match`/`case` can become:

```scala
val inchwormMeasuringTheMarigolds = Array(
    plus.tupled( Seq(1, 1).toTuple ),
    plus.tupled( List(2, 2).toTuple ),
    plus.tupled( Stream(4, 4).toTuple ),
    plus.tupled( scala.collection.mutable.ListBuffer(8, 8).toTuple ),
    plus.tupled( Vector(16, 16).toTuple ))
// inchwormMeasuringTheMarigolds: Array[Int] = Array(2, 4, 8, 16, 32)
```

This is much better.

## Limitations of this approach

This approach does have some limitations:

### Why up to Tuple22?

It turns out that in Scala, functions and Tuples can have up to 22 arguments.  It's in the current language design.

### Downcasts are dangerous

Using this implementation, if the tuple we build has a different arity than the function's own arity, we will get an error at runtime.

```scala
plus.tupled(Seq(3, 1, 2).toTuple)
// java.lang.ClassCastException: scala.Tuple3 cannot be cast to scala.Tuple2
// 	at repl.Session$App$$anonfun$11.apply$mcI$sp(implicits-pt3-implicit-classes2.md:84)
// 	at repl.Session$App$$anonfun$11.apply(implicits-pt3-implicit-classes2.md:84)
// 	at repl.Session$App$$anonfun$11.apply(implicits-pt3-implicit-classes2.md:84)
```

