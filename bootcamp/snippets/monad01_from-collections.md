### Prerequisites:

* [Functions can return functions](/bootcamp/snippets/functions-returning-functions.md)
* [On Iterating without 'Looping'](/bootcamp/snippets/functions-and-loops.md)

--------

# (Mostly) Mathless Monads

> [open on suburban kitchen, Wife and Husband arguing.]
> 
> Wife: New Shimmer is a floor wax!
> 
> Husband: No, new Shimmer is a dessert topping!
> 
> Wife: It’s a floor wax!
> 
> Husband: It’s a dessert topping!
> 
> Spokesman: [ enters quickly] Hey, hey, hey, calm down, you two. New Shimmer is both a floor wax and a dessert topping! Here, I’ll spray some on your mop.. [ sprays Shimmer onto mop] ..and some on your butterscotch pudding. [ sprays Shimmer onto pudding] [ Husband eats while Wife mops]
> 
> Husband: Mmmmm, tastes terrific!
> 
> Wife: And just look at that shine! But will it last?
> 
> Spokesman: Hey, outlasts every other leading floor wax, 2 to 1. It’s durable, and it’s scuff-resistant.
> 
> Husband: And it’s delicious!

Like **Shimmer** from the [Saturday Night Live sketch excerpted above](https://www.nbc.com/saturday-night-live/video/shimmer-floor-wax/n8625), Monads live a dual life.  Unlike **Shimmer**, they're both tasty *and* good for you.

How?

On one hand, the term "Monad" comes from Category Theory in mathematics.  Because the engineering ideas based on them come from math, we can expect them to be durable and scuff-resistant even as engineering trends change.

Some here might reasonably worry that one would have to understand Category Theory to use Monads (and that might not be very tasty).

But it turns out that this is untrue.  Like *Shimmer,* one doesn't need to understand all the underlying mathematical ingredients for it to be delicious.  And good for you too!

In exactly what ways are Monads delicious?  Here are a few:

## Why should I care about Monads to begin with?

Monads are a design pattern.  Similar to how many computer programmers created observers using function pointers, function objects, or similar language features long before the Observer Pattern was described, Monads were implemented over and over again long before Philip Wadler described how they apply to engineering in 1990.

And like Observers (or any other design pattern we now take for granted), as more developers come to understand Monads as a design pattern we will reap similar benefits.

It's these pragmatic benefits I'm after here.  There are many more as you understand this pattern more deeply, but my goal is to help you understand enough to start bridging between Scala's APIs and the more general meaning.

How will we do that?

We will start from the APIs for `#map` and `#flatMap` and understand:

1. The set of things these APIs let us do using the simplest kind of monads--collection monads.
2. How, even with this constraint, monads appear over and over as a general pattern expressing for expressing computation itself.

## Functors

In Scala, `functors` are things that implement `map` and satisfy some math laws we'll get to in a bit.  Why are functors important?  Functors encode 1:1 transformations.

```scala
(1 to 5).map( _*5 )
// res0: collection.immutable.IndexedSeq[Int] = Vector(5, 10, 15, 20, 25)
```

The type signature of its transformation function is `A => B` (or an `A` maps to a `B`).

As a practical matter, that's about it.

## Monads

In Scala, `monads` are things that implement `flatMap` and satisfy some math laws we'll get to in a bit.  Why are monads important?  Monads encode 1:0-n mappings.

That is, for every element present in the input there may be zero corresponding elements present in the output, one corresponding element present in the output, or 2-n corresponding elements present in the output.  To illustrate:

#### 1:0

```scala
(1 to 3).flatMap( _ => Vector() )
// res1: collection.immutable.IndexedSeq[Nothing] = Vector()
```

#### 1:1

```scala
(1 to 3).flatMap( x => Vector(x*5) )
// res2: collection.immutable.IndexedSeq[Int] = Vector(5, 10, 15)
```

#### 1:n (where n=2)

```scala
(1 to 3).flatMap( x => Vector(x, x) )
// res3: collection.immutable.IndexedSeq[Int] = Vector(1, 1, 2, 2, 3, 3)
```

This property of `flatMap` flows naturally from the type signature of `flatMap`'s transformation function: `A => M[B]` where any number of the resulting `B` may be contained inside the result monad `M[B]`.

I guarantee that you've seen this pattern before (though maybe not in this form), and it's worth stopping here to consider where--and why that's significant.

## So what?

Why are functors and monads important?

Together, they come pretty close to a general model of computation itself.  Any random calculation may produce no value, may produce a single value, or may produce many values for a single input.  Some mappings are strictly 1:1; others are 1:0-n transformations.  Named monads or not, this concept itself repeats over and over again in computing.

* Within programs, functions return a result or return 0-n results.
* In relational databases we express 1:1 and 1:many mappings via shared keys.  These are examples of functors and monads respectively, even though that terminology never appears in relational calculus.
* A program interpreter repeatedly transforms a program state S(n) through expressions (functions), each producing a new program state S(n+1) that may contain fewer, equal, or more values than the prior state.
* Apache Spark is just monadic operations over very large streaming data sources/sinks.
* Unix pipes transform sequences of lines, where an individual line from the input file may be transformed to zero lines, one line, or many lines.
* Regular expression operators match single characters or 0-n characters.
* Similaraly, grammars match single or 0-n expressions when parsing.  (Scala's parser library is implemented using monads.)

It all starts by becoming comfortable with `map` and `flatMap` and being aware that these concepts exist elsewhere, but often with less formality and under different names.

In this sense, functors and monads are [programming patterns](https://en.wikipedia.org/wiki/Design_pattern) in the [Gang of Four](https://en.wikipedia.org/wiki/Design_Patterns) sense of the term.

They provide a common vocabulary for communicating with other programmers about problems that follow a similar structure.  And they empower a developer to be able to recognize similar structures and subsequently to code generic solutions for them--avoiding unnecessarily repeating oneself.

## Those math laws

Mostly you don't have to care; mostly the laws state what one would expect from `map` and `flatMap` anyway.  But in case you do care...

1. Functors and monads both respect identity for the objects you put in and get out.
2. Monad (flatMap) operations respect associativity.

As a practical matter, here's what this means:

### Identity

Identity says "if I box an `x` and ask the functor or monad to unbox its contents, I'll get `x` back."

Yeah, programmers have had this expectation of boxed types without thinking about it ever since they learned how to use arrays.  Yawn.

### Associativity

Associativity over `flatMap` means that one can refactor chained flatMap operations as follows:

```scala
m.flatMap(g).flatMap(f) == m.flatMap( x => g(x).flatMap(f) )
```

### Is the associativity law useful?

Yes, generally when considering program refactorings.

Here's one specific example:

Suppose the initial container `m` is an abstraction over a very large stream that is being materialized from a network connection, processed, and the result of the processing pipeline streamed somewhere else.

In this case:

```scala
m.flatMap(g).flatMap(f)
```

would buffer the entire stream (represented by `m`) on the current machine while processing `m.flatMap(g)`.  Then it would process `.flatMap(f)`.

To illustrate, let's make m a Seq[String] and break down the calculation above into its intermediate steps.  First we will define dummy implementations for `g` and `f` and an `m` containing some test data.

```scala
def g(s: String) = Seq(s.toUpperCase)
def f(s: String) = s.split(" ")

val m = Seq("First row", "Second row", "...", "(lots more rows)", "...", "nth row")
// m: Seq[String] = List(
//   "First row",
//   "Second row",
//   "...",
//   "(lots more rows)",
//   "...",
//   "nth row"
// )
```

Now we'll execute `m.flatMap(g).flatMap(f)`, but with the individual expressions separated so we can see the intermediate results.

```scala
val firstExpression = m.flatMap(g)
// firstExpression: Seq[String] = List(
//   "FIRST ROW",
//   "SECOND ROW",
//   "...",
//   "(LOTS MORE ROWS)",
//   "...",
//   "NTH ROW"
// )
val result1 = firstExpression.flatMap(f)
// result1: Seq[String] = List(
//   "FIRST",
//   "ROW",
//   "SECOND",
//   "ROW",
//   "...",
//   "(LOTS",
//   "MORE",
//   "ROWS)",
//   "...",
//   "NTH",
//   "ROW"
// )
```

As we can see, this structure processes all data inside `m` through `g` before piping that entire `firstExpression` through `f`.

In this example `m` isn't large enough for the results inside `firstExpression` to be problematic.

But imagine if `m` were a large sequence of data read from a network connection.

In that case it would at best be inefficient to process all of `m` through the initial expression in the pipeline, process the entire result through the next expression, the next one after that, and so on.  Worse, given enough source data in the original `m`, a pipeline structured like this would be unable to complete without running out of memory.

The associativity law tells us that we can refactor our algorithm from a single flat pipeline to the following:

```scala
m.flatMap( x => g(x).flatMap(f) )
```

This runs subsequent steps repeatedly on each individual element from `m`, one at a time, incrementally producing the entire pipeline's results.  This eliminates the need to buffer the result of the entire initial transformation before continuing to the second transformation, and so on; it fixes the memory bottleneck while returning the same result.

```scala
val result2 = m.flatMap( row => g(row).flatMap(f) )
// result2: Seq[String] = List(
//   "FIRST",
//   "ROW",
//   "SECOND",
//   "ROW",
//   "...",
//   "(LOTS",
//   "MORE",
//   "ROWS)",
//   "...",
//   "NTH",
//   "ROW"
// )

result2 == result1
// res4: Boolean = true
```


#### A final note on associativity

Monad associativity only governs the `flatMap` operation itself, not the `f` or `g` operations.

Here's an example illustrating this:

Integer subtraction isn't associative.  So, let's test `flatMap` associativity (using a boxed Integer type) where the `f` and `g` functions both subtract.

```scala
3-2-1 == 3-(2-1)
// res5: Boolean = false

{
  def minus2(i: Int) = Integer(i-2)
  def minus1(i: Int) = Integer(i-1)

  Integer(3).flatMap(minus2).flatMap(minus1) ==
    Integer(3).flatMap { x =>
      minus2(x).flatMap(minus1)
    }
}
// res6: Boolean = true
```

## Epilogue: Monads Really Aren't Scary!

In 1990, Philip Wadler published his initial research paper explaining how Monads from Abstract Algebra / Category Theory are useful to programmers, optimistically titled *[Comprehending Monads](https://ncatlab.org/nlab/files/WadlerMonads.pdf)*.  A few months and a lot of scratched heads later, another researcher whose name is apparently only preserved as "Frederik" wrote a further paper titled, *[Comprehending 'Comprehending Monads'](http://talks.cam.ac.uk/talk/index/6732)*

It didn't help that in Haskell, the monad operator is written as:

```haskell
>>=
```

Obviously.

The unhappy result was that for a time, blog articles and presentations proliferated where various people used multitudinous methods attempting to connect Category Theory with Programming Practice.  These attempts were largely unsuccessful.

Here's a brief and memorable sample:

* [Monads are like burritos](https://blog.plover.com/prog/burritos.html)
* [Monads are not burritos](https://neoeinstein.github.io/monads-are-not-burritos/#/)
* Monads are elephants: [part 1](http://james-iry.blogspot.com/2007/09/monads-are-elephants-part-1.html), [part 2](http://james-iry.blogspot.com/2007/10/monads-are-elephants-part-2.html), [part 3](http://james-iry.blogspot.com/2007/10/monads-are-elephants-part-3.html), [part 4](http://james-iry.blogspot.com/2007/11/monads-are-elephants-part-4.html) (Note: I actually liked this series quite a bit.)

The predictable consequence of profusely proliferating abstract articles mangling monads was that many engineers reasonably concluded that monads must simply be incomprehensible to those without a graduate-level background in mathematics.

But now `map` and `flatMap` (or whatever they happen to be named in a given language's libraries) are fairly widely understood.

And hopefully this change in our shared collective consciousness has opened a new avenue toward a deeply pragmatic, yet Rigorous Enough explanation that many if not most modern engineers can now travel toward a realistic and useful working programmer's understanding of this previously-challenging topic.


## Some things to try

Monads are like variables in a programming language.  They are containers for a value and/or an operation (function).

`flatMap` does all the things a general calculation can do:

* Transform its input to an output of the same arity (number of elements).
* Transform its input to an empty output.
* Transform its input to an output with a lesser arity.
* Transform its input to an output with a greater arity.

Using this principle, one can imagine a programming language interpreter defined in terms of monads as variables and `flatMap` operations as calculations.

Try it!  Can you implement pipelines that branch depending on one or more conditions using monads?  How about loops?  See how far you can get.
