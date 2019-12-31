# Introduction

This article is a deep dive into when `for` comprehensions are more expressive and easier to read than
`flatMap`/`map` and how to refactor from chained/nested `flatMap`s and `map`s to `for` comprehensions.

We'll start with really basic use-cases that will be familiar to anyone who knows object-oriented languages
like Java or C# and build skills until we can refactor complex nested mapping operations into much more
expressive `for` comprehensions.

The benefits are:

* A clearer understanding of how `for` expressions actually work under-the-hood.
* More fluency working with both mapping operations and `for` comprehensions.

# Of maps and comprehensions

Coming to the Scala language, you've probably heard that `for` expressions are actually syntactic sugar,
mostly over `flatmap` and `map`.  And you might have run into explanations
like the one on the [tutorials/FAQ](https://docs.scala-lang.org/tutorials/FAQ/yield.html) page
at scala-lang.org giving several examples of how various `for` expressions<sup>[1](#Footnotes)</sup> are actually
translated into "multiple monadic operations" by Scala's compiler.<sup>[2](#Footnotes)</sup>

And even if all of the examples on a site like the one above made sense to you, at least in theory,
they may not have actually helped communicate how and when a programmer can rewrite a sequence of
`flatmap` and `map` operations as a `for` comprehension, even though the simple example provided at
the end of tutorials like the one above does supply a hint that for more complicated code, `for`
comprehensions can wind up being more readable.  To quote from the tutorial above:

```scala
l.flatMap(sl => sl.filter(el => el > 0).map(el => el.toString.length))

// can be translated to:

for{
  sl <- l
  el <- sl
  if el > 0
} yield el.toString.length
```

Obviously.  Right.  Okay...

When I was learning Scala, examples like the above didn't help me understand how and when
it would be to my advantage to rewrite nested `flatmap`s as a `for` comprehension.
That's the first thing I'll attempt to demystify here.

Secondly, a common trait of code written using an IDE and relying on nested `flatMap` and `map`
operations is that the mapping operations can tend to become excessively nested, tangled, harder to
reason about, and also harder to see how to refactor to make clearer.  After explaining how to
translate nested mapping operations, I'll show how to apply this skill to refactor deeply nested
and somewhat tangled code to a considerably cleaner, more understandable state.

We'll start by comparing `for` comprehensions with loops in other languages.

## For the love of loops?

Coming from most other languages, a freshly-minted Scala engineer would naturally be forgiven if he or she
expected that all `for` expressions necessarily loop over collections' contents.  After all, that's
what they do in other languages, right?

Actually, they do in Scala too, but only if one relaxes and/or abstracts one's definition of what
constitutes a *collection*.

### Let's loop over Ints!

```scala
val xs = 2 to 8
// xs: Range.Inclusive = Range.Inclusive(2, 3, 4, 5, 6, 7, 8)
for (x <- xs) yield Math.pow(2, x)
// res0: collection.immutable.IndexedSeq[Double] = Vector(
//   4.0,
//   8.0,
//   16.0,
//   32.0,
//   64.0,
//   128.0,
//   256.0
// )
```

So, yeah, we can compute powers of 2.  How about array lookups too?

(Note: If you can't see some or all of the emoji used here, the Noto Mono Sans font supports all of them.)

```scala
val animals = Array("🐕", "🐈", "otter", "🐒", "bison", "🐦", "parakeet", "🦛", "🐛", "locust")
// animals: Array[String] = Array(
//   "\ud83d\udc15",
//   "\ud83d\udc08",
//   "otter",
//   "\ud83d\udc12",
//   "bison",
//   "\ud83d\udc26",
//   "parakeet",
//   "\ud83e\udd9b",
//   "\ud83d\udc1b",
//   "locust"
// )

for (x <- xs) yield animals(x)
// res1: collection.immutable.IndexedSeq[String] = Vector(
//   "otter",
//   "\ud83d\udc12",
//   "bison",
//   "\ud83d\udc26",
//   "parakeet",
//   "\ud83e\udd9b",
//   "\ud83d\udc1b"
// )
```

Yawn.

But wait...

# Transforming and mapping

Scala developers also learn early-on that collection classes define methods like `map` and `flatmap`
that can be used for a similar purpose to the examples above.  Here's the powers-of-two example again:

```scala
xs.map(Math.pow(2, _))
// res2: collection.immutable.IndexedSeq[Double] = Vector(
//   4.0,
//   8.0,
//   16.0,
//   32.0,
//   64.0,
//   128.0,
//   256.0
// )
```

In this situation, few Scala developers would argue that the `for` expression above is easier to read than
this `map` example.<sup>[3](#Footnotes)</sup>

Therefore, in what cases would `for` expressions be easier?

Just as with other languages, `for` expressions start to prove their worth when they become nested.
For example, the following two expressions are equivalent, but most developers would consider the `for`
expression to be easier to comprehend.

```scala
{
  val powerTable1 = (2 to 3).flatMap {
    y => (3 :: 4 :: Nil).map {
      x => (x, y)
    }
  }.map {
    case (x, y) => List(x, y, Math.pow(x, y))
  }

  val powerTable2 = for {
    y <- 2 to 3
    x <- 3 :: 4 :: Nil
  } yield List(x, y, Math.pow(x, y))

  powerTable1 == powerTable2
}
// res3: Boolean = true
```

When coding with `Int`s and `Double`s, most Scala developers would readily apply a `for` expression
here--instead of nesting map and flatmap operations--because it's what we're used to doing with other
languages.

This leads us to the question:  Can we leverage our shared intuition about how `for` loops work in other
languages to help us use them more effectively in Scala?  Let's see.

## How do we recognize `for` loops elsewhere?

If a programmer was confronted with code like the nested and chained `flatMap` and `map` example above, how
would she recognize that the code could be readily translated to a much-more-readable `for` expression?

To most developers, the answer is probably very similar.

First, the developer would notice that the nested operations iterate over collections of a similar type
(in this case, both an IndexedSeq; in Java, an Iterable).

```scala
val powerTable2 = (2 to 3).flatMap {   // Range is an IndexedSeq
  y => (3 :: 4 :: Nil).map {           // List is an IndexedSeq
    x => (x, y)
  }
}.map {
  case (x, y) => List(x, y, Math.pow(x, y))
}
```

Then, noting the similarities, he would then "unwind" the mapping operations into a `for` expression that
is much easier to read.

```scala
val powerTable1 = for {
  y <- 2 to 3                          // Range
  x <- 3 :: 4 :: Nil                   // List
} yield List(x, y, Math.pow(x, y))
```

The result of both expressions is an `IndexedSeq[List[Double]]`.  Why?  The resulting outer collection
(`IndexedSeq`) is the common collection type between the two input collections' type hierarchies.  The
inner `List[Double]` is the final result we computed.

From these observations, we can begin expressing initial rules helping us to identify nested mapping
expressions that can be translated into `for` comprehensions:

* The body of the `for` comprehension must have the same type--or a common superclass--on the right
  side of the `<-`.
* The resulting collection will be the most specific common superclass of all expressions on the right-hand
  side of the `<-`.
* The outer collection operations in the nested structure must be `flatMap`(s) and the innermost one a
  `map` operation.

The last principle might not be obvious yet, so let's compare our nested `flatMap`/`map` expression above
with one that purely `map`s and see what results we get:

```scala
(2 to 3).flatMap {
  y => (3 :: 4 :: Nil).map {
    x => (x, y)
  }
}
// res4: collection.immutable.IndexedSeq[(Int, Int)] = Vector(
//   (3, 2),
//   (4, 2),
//   (3, 3),
//   (4, 3)
// )

(2 to 3).map {
  y => (3 :: 4 :: Nil).map {
    x => (x, y)
  }
}
// res5: collection.immutable.IndexedSeq[List[(Int, Int)]] = Vector(
//   List((3, 2), (4, 2)),
//   List((3, 3), (4, 3))
// )
```

When the outer nesting levels use `map`, the intermediate data structures are retained.  This makes it
impossible for the `yield` expression to compute a result in the way we would expect a "looping" construct to do.

To make the point even clearer, let's consider an example with three nesting levels illustrating

1. Code we cannot translate to a `for` expression (and probably isn't what was intended to begin with).
2. Code we can translate to a `for` expression.
3. The resulting `for` expression after translation.

```scala
//// (1) This cannot be translated to a `for`
val noForAllowed = Vector(3, 1).map {
  z => Array(2).map {
    y => List(4).map {
      x => (x, y, z)
    }
  }
}

//// (2) This can be translated to a `for`
val translatable = Vector(3, 1).flatMap {
  z => Array(2).flatMap {
    y => List(4).map {
      x => (x, y, z)
    }
  }
}

//// (3) After translation
val forComprehension = for {
  z <- Vector(3, 1)
  y <- Array(2)
  x <- List(4)
} yield (x, y, z)
```

Why?  Let's look at the results:

```scala
noForAllowed
// res6: Vector[Array[List[(Int, Int, Int)]]] = Vector(
//   Array(List((4, 2, 3))),
//   Array(List((4, 2, 1)))
// )

translatable
// res7: Vector[(Int, Int, Int)] = Vector((4, 2, 3), (4, 2, 1))

forComprehension
// res8: Vector[(Int, Int, Int)] = Vector((4, 2, 3), (4, 2, 1))

noForAllowed == translatable
// res9: Boolean = false

translatable == forComprehension
// res10: Boolean = true
```

To reiterate: nested structures that can be translated to `for` expressions must use `flatMap` as their
outermost "looping" structure and a `map` for the innermost expression.

# Generalizing collections

So far we've examined the following:

* For comprehensions over regular collections.
* When `flatmap`/`map` patterns are better expressed as `for` comprehensions.

What if the thing we are "looping" over isn't a traditional collection but still supports `map` and
`flatmap`?  Can we use a `for` comprehension to pull data out of these kinds of things?  Let's reexamine
the criteria we discussed earlier for when we can translate nested `flatmap`/`map`s into a `for`
comprehension; can these criteria still apply for things that support `flatmap`/`map` but aren't
traditional collections?

* The body of the `for` comprehension must have the same type--or a common superclass--on the right
  side of the `<-`.
* The resulting collection will be the most specific common superclass of all expressions on the right-hand
  side of the `<-`.
* The initial collection operations in the nested structure must be a `flatMap` and the innermost one a
  `map` operation.

Yes these criteria still apply, so yes we can use `for` expressions with non-collection "collections" too.
Sound strange?  Let's examine some concrete examples:

## Options to consider

Some people don't really consider `Option[T]` a collection.  Others consider it a collection containing
0 or 1 element.  No matter where you might fall in this debate, since `Option[T]`s support `map` and `flatMap`,
we can still use them with `for` comprehensions.  If we do, how do `Option[T]`s behave inside a `for`
expression and does this still improve on nested `flatMap`/`map`?

Here's a small example:

```scala
{
  // Define some stub service methods

  def getIdFromFirstService(): Option[String] = Some("databaseTable1Id")
  def getIdFromSecondService(): Option[String] = Some("databaseTable2Id")

  def queryTheDatabase(id1: String, id2: String): Option[(String, String)] = {
    Some { ("TICKET-42", "Calculate the meaning of life") }
  }

  // Nested flatmaps can unpack multiple Option[T] results
  val queryResult1 = getIdFromFirstService().flatMap { firstServiceId =>
    getIdFromSecondService().flatMap { secondServiceId =>
      queryTheDatabase(firstServiceId, secondServiceId).map { result =>
        result
      }
    }
  }

  // The above nesting conforms to our comprehension translation rules, so...
  val queryResult2 = for {
    firstServiceId <- getIdFromFirstService()
    secondServiceId <- getIdFromSecondService()
    result <- queryTheDatabase(firstServiceId, secondServiceId)
  } yield result

  // ...as before, the results are the same
  queryResult1 == queryResult2
}
// res11: Boolean = true
```

Even for this very simple example, most people would consider the `for` expression to be more readable.
And as before, the more complicated and/or deeply nested the `flatMap`/`map` blocks get, the more
readable the `for` expression becomes compared to nested `flatMap` and `map` blocks.

From the nested flatMap example we translated, you can probably guess how the `for` comprehension
behaves if either of the first services returns `None`, but let's try it in the REPL to be sure.

We'll start by having the first service always return `None`, and we'll `println`-log when each thing
is called so we can see exactly what happens in what order.

```scala
{
  def getIdFromFirstService(): Option[String] = {
    println("Calling the first service")
    None
  }

  def getIdFromSecondService(): Option[String] = {
    println("Calling the second service")
    Some("databaseTable2Id")
  }

  def queryTheDatabase(id1: String, id2: String): Option[(String, String)] = {
    println("Querying the database")
    Some { ("TICKET-42", "Calculate the meaning of life") }
  }

  for {
    firstServiceId <- getIdFromFirstService()
    secondServiceId <- getIdFromSecondService()
    result <- queryTheDatabase(firstServiceId, secondServiceId)
  } yield result
}
// Calling the first service
// res12: Option[(String, String)] = None
```

Let's note that the `for` comprehension behaves just like the prior nested `flatMap`/`map` example.

Here's how the nested `flatMap` example works:

* The nested `flatMap` example first calls `getIdFromFirstService`, which returns `None`.
* `None` contains no value, so there is nothing to pass to the `flatMap` callback.
* Consequently, the initial `flatMap` never calls its callback, but instead returns `None`
  as the value of the entire expression.

The `for` comprehension works exactly the same way.

* First, the `for` comprehension calls `getIdFromFirstService`, which returns `None`.
* Since `None` contains no value, there is nothing to assign to `firstServiceId`.
* Consequently, the `for` expression stops evaluating at this point and returns `None` as
  the value of the entire expression.

This behavior is identical to the behavior of any empty collection calling `flatMap` or `map`;
the value of the entire expression results in an empty collection of the same type as the initial
collection.

Now let's examine the behavior if the second service returns `None`:

```scala
{
  def getIdFromFirstService(): Option[String] = {
    println("Calling the first service")
    Some("databaseTable1Id")
  }

  def getIdFromSecondService(): Option[String] = {
    println("Calling the second service")
    None
  }

  def queryTheDatabase(id1: String, id2: String): Option[(String, String)] = {
    println("Querying the database")
    Some { ("TICKET-42", "Calculate the meaning of life") }
  }

  for {
    firstServiceId <- getIdFromFirstService()
    secondServiceId <- getIdFromSecondService()
    result <- queryTheDatabase(firstServiceId, secondServiceId)
  } yield result
}
// Calling the first service
// Calling the second service
// res13: Option[(String, String)] = None
```

In this situation, the first service is called and returns `Some("databaseTable1Id")`, so
processing continues and calls the second service. But this one returns `None`, and as there is
no value to assign `secondServiceId`, the entire `for` comprehension returns `None`.

These examples illustrate how operations on things that aren't collections in the traditional sense
(but that support `flatMap` and `map`) can also be used inside `for` comprehensions; as before,
the `for` expression normally improves code readability compared with nesting `flatMap` and `map`.

## "Loop" to the future

Let's consider a final example of a type that definitely isn't a collection in the traditional sense,
but that supports `flatMap`/`map` and so can still be used inside `for` comprehensions:
`Future[T]`.<sup>[4](#Footnotes)</sup>

First let's consider how this can work since the result of a single `Future[T]` operation isn't
necessarily known right away, much less the result of multiple chained `Future[T]` operations.

```scala
someFuture.flatMap {    // returns Future(result) when calculation is complete
  result => Future.successful(result)
}
someFuture.map {        // returns Future(result) when calculation is complete
  result => result
}
```

The above example illustrates how, just like Option, applying `flatMap` or `map` to a `Future`
calls the `Future`'s callback with the value produced by the `Future`'s calculation, which can be
viewed as the value "inside" the `Future`.  Unlike Option, `Future` only fails to return a value
if the background calculation failed.

Even with the differences outlined above, one can consider a `Future` to be a container for a value
that might not be available immediately.  Calling `map` or `flatMap` on a `Future` simply
blocks, then calls the mapping operation's callback when the result becomes available.

The result is that we can translate nested `flatMap`/`map` operations over `Future[T]` into
`for` expressions too.  Let's try!

In this example we'll define some new service methods, this time with the second service accepting
the result from calling the first service:

```scala
def getFromFirstService(): Future[String] = Future { "serviceResult" }
def getIdFromSecondService(serviceResult: String): Future[String] = Future { "databaseTable" }
def queryTheDatabase(id: String): Future[(String, String)] = Future { ("TICKET-1000", "One Day") }
```

Next, as before we'll define a method to call the services using `flatMap` and `map` to retrieve
results and a second translating the `flatMap`s and `map` to a `for` comprehension
using the principles described above:

```scala
def callUsingFlatmap() = {
  getFromFirstService().flatMap { firstServiceResult =>
    getIdFromSecondService(firstServiceResult).flatMap { id =>
      queryTheDatabase(id).map { result =>
        result
      }
    }
  }
}

def callUsingFor() = {
  for {
    firstServiceResult <- getFromFirstService()
    id <- getIdFromSecondService(firstServiceResult)
    result <- queryTheDatabase(id)
  } yield result
}
```

As before, both approaches return the same results:

```scala
val flatMapResult = Await.result(callUsingFlatmap(), 5.seconds)
// flatMapResult: (String, String) = ("TICKET-1000", "One Day")

val forComprehensionResult = Await.result(callUsingFor(), 5.seconds)
// forComprehensionResult: (String, String) = ("TICKET-1000", "One Day")

flatMapResult == forComprehensionResult
// res14: Boolean = true
```

# One way to avoid writing complicated nested maps in the first place

Actually, when I'm coding I frequently don't identify in advance when nested `flatMap`s could be
better expressed as a `for` comprehension right away.  Sometimes I do, but not always.

What I try to do as a part of writing expressive code is to pay attention to my `flatMap` and
`map` operations.  If they are beginning to follow the two rules expressed previously, I start
refactoring into `for` comprehensions.  To review, those two questions I'm constantly asking are:

* Are all the mapping operations over the same container type or container types with a superclass
  that also implements `flatMap` and `map`?
* Am I writing one or more `flatMap`s followed by a single `map`?

If so, I try to immediately catch myself use a `for` expression instead.

# Refactoring for fun and profit

Writing code that takes advantage of `for` comprehensions helps produce an expressive, easy-to-follow
codebase.  However, what about when one encounters a tangled mess of nested mapping operations in
existing code?

What if the code is so deeply nested that it is impossible to understand without relying on my IDE's
"show this thing's type" command?

What if the nested code is so long that by the time I've read the types to the end of the method I've
forgotten what the types were at the beginning?

Sometimes the I can be sure I'm understanding the code correctly is to refactor it; and in that case, the
code often is tangled enough that refactoring to `for` expressions isn't immediately obvious
or straightforward.

And even if the code isn't so tangled as to be incomprehensible, refactoring nested mapping structures
often significantly improves the code's readability and expressiveness.

Since refactoring complicated nested mapping operations often isn't easy or obvious, I'll conclude
by showing how I approach refactoring nested chained mapping operations based on several real-world
examples.

Here are the steps we'll follow:

* Introduce the initial implementation that uses `flatMap` and `map` excessively and isn't as easy to
  read as it could be.
* Illustrate how to refactor this code to a point where the parts that can be converted to `for`
  comprehensions are much more apparant.
* Refactor to use `for` expressions where possible and useful.

## Initial implementation using `flatMap` and `map`

First, we'll define some model and service stubs.

```scala
case class AuthToken()
case class User()
case class UserDetails()
case class Role()

object credentialService {
  def oauthLogin(oldAuthToken: AuthToken): Future[Option[AuthToken]] =
    Future { Some(AuthToken()) }
  def getUserDetails(token: Option[AuthToken], role: Option[Role]): Future[Option[UserDetails]] =
    Future { Some(UserDetails()) }
}


object securityService {
  def getRole(token: Option[AuthToken]): Future[Option[Role]] =
    Future { Some(Role()) }
}


case class UserPromotion()
case class GenericAd()

object promotionsService {
  def getPromotions(token: AuthToken, userDetails: UserDetails): Future[List[UserPromotion]] =
    Future { List(UserPromotion(), UserPromotion()) }
}

object adService {
  def getAds(token: AuthToken, userDetails: UserDetails): Future[List[GenericAd]] =
    Future { List(GenericAd(), GenericAd(), GenericAd(), GenericAd(), GenericAd()) }
}
```

The code we refactor will log in, obtain the user's role and details (using the stub services above), then finally
query the `promotionService` and `adService` for promotions and ads customized for that user.  Results will
be returned using the following class:

```scala
case class UserModel(
  token: AuthToken,
  details: UserDetails,
  role: Role,
  promotions: List[UserPromotion],
  ads: List[GenericAd]
)
```

Here's the code we'll start with and then refactor:

```scala
object mappingAuthService1 {

  def login(oldAuth: AuthToken): Future[Option[UserModel]] = {

    credentialService.oauthLogin(oldAuth).flatMap { token =>
        securityService.getRole(token) flatMap { role =>
          credentialService.getUserDetails(token, role) map { userDetails =>
            (token, role, userDetails)
          }
        }
      }.flatMap { case (token, role, userDetails) =>
        token.flatMap { t: AuthToken =>
          role.flatMap { r: Role =>
            userDetails.map { u: UserDetails =>
              (t, r, u)
            }
          }
        }.map { case (token, role, userDetails) =>
          promotionsService.getPromotions(token, userDetails) flatMap { promotions =>
            adService.getAds(token, userDetails) map { ads =>
              UserModel(token, userDetails, role, promotions, ads)
            }
          }
        } match {
          case Some(f) => f.map(Some(_))
          case None => Future.successful(None)
        }
    }
  }
}
```

The result of running it is:

```scala
val mappingsAhoy = Await.result(mappingAuthService1.login(AuthToken()), 5.seconds)
// mappingsAhoy: Option[UserModel] = Some(
//   UserModel(
//     AuthToken(),
//     UserDetails(),
//     Role(),
//     List(UserPromotion(), UserPromotion()),
//     List(GenericAd(), GenericAd(), GenericAd(), GenericAd(), GenericAd())
//   )
// )
```

## Goals from refactoring the above code

When I encounter code like this, my first thought is that I have to use my IDE to understand how types are transformed through this `login` method.

My second thought is often that, "the method tries to do too many things, so by the time I'm at the end I may have forgotten the types that were used at the beginning."

What we want in our end result is:

* Obvious types for the input and output of each transformation.
* Methods that perform exactly one operation.
* `for` comprehensions used when they enhance readability.

## IDE tooling is nice but obvious types are nicer

The first thing I usually do in order to help myself understand code like this is to refactor to make the types obvious at key places in the code.  To do this I use the following strategies:

* Break up chained/nested operations into discrete steps with variables naming and explaining each step.
* If a variable is a boxed type that is nested only one level, e.g.: `A[T]`, and there is a way to name
  it so that the variable's type is obvious in context, I'll just rename the variable.  For example:
  I might rename an `Option[AuthToken]` from `token` to `maybeToken`.
* Sometimes renaming a variable doesn't make its type in the code obvious or gets long and cumbersome.
  (For example, `maybeFutureOptionListFoobar` isn't a great name.)  In this situation I'll add a normal
  type annotation where the variable is initially declared.
* Occasionally I'll add a comment noting the type of the container being `flatMap`'d or `map`'d.

Also, Sometimes nested and/or chained structures are deep enough that it's hard to follow what each step does without relying on IDE tooling to know the exact types at key steps.

In this situation, I'll break up the expression and introduce intermediate variables that are either named obviously or that have type annotations.  Also, if an expression is really a generic operation, I'll extract this to its own method.

Refactoring our code above using the above principles (and adding white space in strategic places) results in the following:

```scala
object mappingAuthService2 {

  def login(oldAuth: AuthToken): Future[Option[UserModel]] = {

    val futureAuthDetails: Future[(Option[AuthToken], Option[Role], Option[UserDetails])] =
      credentialService.oauthLogin(oldAuth) flatMap { maybeToken =>
        securityService.getRole(maybeToken) flatMap { maybeRole =>
          credentialService.getUserDetails(maybeToken, maybeRole) map { maybeUserDetails =>
            (maybeToken, maybeRole, maybeUserDetails)
          }
        }
      }

    futureAuthDetails flatMap { case (maybeToken, maybeRole, maybeUserDetails) =>
      val maybeAuthDetails: Option[(AuthToken, Role, UserDetails)] = maybeToken flatMap { token =>
        maybeRole flatMap { role =>
          maybeUserDetails map { userDetails: UserDetails =>
            (token, role, userDetails)
          }
        }
      }

      val boxedUserModel: Option[Future[UserModel]] = maybeAuthDetails map { case (token, role, userDetails) =>
        promotionsService.getPromotions(token, userDetails) flatMap { promotions =>
          adService.getAds(token, userDetails) map { ads =>
            UserModel(token, userDetails, role, promotions, ads)
          }
        }
      }
      optionFuture2FutureOption(boxedUserModel)
    }
  }

  private def optionFuture2FutureOption[T](optionFuture: Option[Future[T]]): Future[Option[T]] = {
    optionFuture match {
      case Some(future) => future.map(Some(_))
      case None => Future.successful(None)
    }
  }
}
```

The results of this refactoring are:

* Each step in the transformation is now obvious without requiring IDE tooling.
* By simply scanning the code, we can now identify `flatMap, flatMap, map` sequences to translate into
  `for` comprehensions.

Let's apply the `for` comprehension translation rules next.

```scala
object mappingAuthService3 {

  def login(oldAuth: AuthToken): Future[Option[UserModel]] = {

    val futureAuthDetails: Future[(Option[AuthToken], Option[Role], Option[UserDetails])] = for { // over Future
      maybeToken <- credentialService.oauthLogin(oldAuth)
      maybeRole <- securityService.getRole(maybeToken)
      maybeUserDetails <- credentialService.getUserDetails(maybeToken, maybeRole)
    } yield (maybeToken, maybeRole, maybeUserDetails)

    futureAuthDetails flatMap { case (maybeToken, maybeRole, maybeUserDetails) => // over Future
      val maybeAuthDetails: Option[(AuthToken, Role, UserDetails)] = for { // over Option
        token <- maybeToken
        role <- maybeRole
        userDetails <- maybeUserDetails
      } yield (token, role, userDetails)

      val boxedUserModel: Option[Future[UserModel]] = maybeAuthDetails map { case (token, role, userDetails) =>
        for { // over Future
          promotions <- promotionsService.getPromotions(token, userDetails)
          ads <- adService.getAds(token, userDetails)
        } yield UserModel(token, userDetails, role, promotions, ads)
      }

      optionFuture2FutureOption(boxedUserModel)
    }
  }

  private def optionFuture2FutureOption[T](optionFuture: Option[Future[T]]): Future[Option[T]] = {
    optionFuture match {
      case Some(future) => future.map(Some(_))
      case None => Future.successful(None)
    }
  }
}
```

As expected, converting nested mapping operations into `for` comprehensions improved our code's
readability.

Some will want to stop here, but there is one last significant opportunity to refactor this code
that is easier to spot now:

Notice that the initial `for` expression in the `login` method "loops" over `Future`, so its result
(`futureAuthDetails`) is also a `Future`.  The code then immediately `flatMap`s over
`futureAuthDetails`.

It seems like we ought to be able to move the second `flatMap` operation inside the initial `for`
comprehension, since both the `flatMap` and the initial `for` operate over `Future`.

Can we?

It turns out that we can, but if we just copypasta the `flatMap` block into the initial `for` and add
punctuation to make it compile, that initial `for` will itself become deeply nested and harder to follow.

However, if we extract each discrete operation out of `login` into individual methods, then we
can easily move the second `flatMap` functionality into the initial `for` comprehension.  How?
The method extracted from the body of the `futureAuthDetails.flatMap {...}` operation can be
now added to the end of the initial `for`.

```scala
object mappingAuthService4 {

  def login(oldAuth: AuthToken): Future[Option[UserModel]] = for { // over Future
    maybeToken <- credentialService.oauthLogin(oldAuth)
    maybeRole <- securityService.getRole(maybeToken)
    maybeUserDetails <- credentialService.getUserDetails(maybeToken, maybeRole)
    maybeUserModel <- optionFuture2FutureOption( computeUserModel(maybeToken, maybeRole, maybeUserDetails) )
  } yield maybeUserModel

  private def computeUserModel(
    maybeToken: Option[AuthToken],
    maybeRole: Option[Role],
    maybeUserDetails: Option[UserDetails]
  ): Option[Future[UserModel]] = for { // over Option
    token <- maybeToken
    role <- maybeRole
    userDetails <- maybeUserDetails
  } yield {
    addPromotionsAndAds(token, role, userDetails)
  }

  private def addPromotionsAndAds(token: AuthToken, role: Role, userDetails: UserDetails): Future[UserModel] = {
    for { // over Future
      promotions <- promotionsService.getPromotions(token, userDetails)
      ads <- adService.getAds(token, userDetails)
    } yield UserModel(token, userDetails, role, promotions, ads)
  }

  private def optionFuture2FutureOption[T](optionFuture: Option[Future[T]]): Future[Option[T]] = {
    optionFuture match {
      case Some(future) => future.map(Some(_))
      case None => Future.successful(None)
    }
  }
}
```

For final fun, let's prove that the final refactored version behaves the same as the initial nested `flatMap`/`map` code:

```scala
mappingsAhoy == Await.result(mappingAuthService4.login(AuthToken()), 5.seconds)
// res15: Boolean = true
```

At the beginning of this exercise we set out to transform the code so that it has:

* Obvious types for the input and output of each transformation.
* Methods that each perform exactly one operation.
* `for` comprehensions used when they enhance readability.

Arguably, we've done it, and the result is cleaner and easier to follow.

# Conclusion

Hopefully if you came to this article without a clear understanding of how `for` comprehensions work, that you now have a clearer understanding of places they can significantly improve readability.  Scala's `for` expressions support more features than described here, but as you gain confidence with the basic skills presented here you will be able to move on to more advanced features.

And I hope you are beginning to see how to use `for` expressions to write cleaner, easier-to-follow code compared with the equivalent algorithm expressed using nested mapping operations.

## Recap

Except for the most trivial examples, a `for` expression is usually more readable than the equivalent nested mapping expressions.  Because of this, writing and/or refactoring nested mapping code to use `for` comprehensions is normally a good practice.

The two principles that identify when nested mapping code can be translated into `for` comprehensions are:<sup>[5](#Footnotes)</sup>

* Nested `flatMap` and `map` operations have a single `map` operation, and it is the
  innermost-nested operation.
* All "containers" over which `flatMap` and `map` operate must be the same class or have a common
  superclass implementing both `flatMap` and `map`.

This third principle describes the output type from a `for` expression.

* Nested `flatMap`/`map` operations that have been translated into a `for` comprehension return a
  result type that is most specific superclass of all containers on the right-hand sides of the `<-` arrow.

We showed how classes that one might not initially describe as collection classes, but that
define `flatMap` and `map` operations, also benefit from increased readability when used in `for`
comprehensions instead of nested mapping operations.

Lastly, we refactored a moderately complicated nested mapping structure, breaking down the steps
involved into:

* Introducing intermediate variables identifying the purpose and types of discrete steps in the
  algorithm.
* Extracting these discrete steps into methods of their own that each do a single thing.
* Identifying opportunities to combine steps involving common types.  In our example we moved a
  lone `flatMap` over a `Future` into a `for` comprehension over `Future`.

The result was code that is significantly easier to read and to reason about without requiring IDE
tool support.

###### Footnotes

1. For expressions are usually referred to as `for` comprehensions, because the are a specific type
   of expression.
2. We're not discussing Monads any more here.  You're welcome.
3. Perhaps the `for` expression is more familiar for developers coming from object-oriented and/or
   imperative languages; in this case it's not intrinsically easier to read.
4. One way of looking at a `Future` as similar to a collection is that a `Future` is a container for
   a value that may or may not have been computed yet.
5. Sometimes code doesn't follow the above principles already, but can be refactored into this structure.
   These situations almost always significantly increase readability when refactoring from nested mapping
   structures to one or more `for` comprehensions.

