# Dynamic Scala

All Scala types cannot be statically checked at compile time and/or recognized independently at runtime.  In the Scala community's culture, the repeated answer has been to avoid these corner cases and instead rely on what can be statically checked at compile time.

While I fully embrace the goal of 100% compile-time type safety, I also believe that we do ourselves a disservice when our zeal for provable program correctness obstructs pragmatic solutions when problems don't easily fit this mold.

Here I advocate for the Scala community to embrace the discomfort that our typecheckers aren't perfect yet and in so doing to discover new approaches to old problems that I believe may offer cleaner solutions than the ones we currently embrace.  And I intend to do this while respecting as many Scala values as possible.  Let's start this conversation.

## Heterogeneous type schlepping

One place Scala's type checking limitations create pain is with heterogeneous types.  For example:

```scala
val sounds: List[String] = List("moo", "oink", "meow", "rrrruff")
// sounds: List[String] = List("moo", "oink", "meow", "rrrruff")
```

typechecks perfectly.  And [Shapeless](https://github.com/milessabin/shapeless) allows us to safely define many kinds of heterogeneous lists as well.  However, a heterogeneous map doesn't fare nearly as well:

```scala
Map('name -> "Cocoa", 'breed -> "mutt", 'age -> 12)
// res0: Map[Symbol, Any] = Map('name -> "Cocoa", 'breed -> "mutt", 'age -> 12)
```

Here we're left with a `Map[Symbol,Any]`, which completely breaks the static type guarantees Scala's compiler normally provides.  And Scala doesn't provide a complete solution to this problem.

The usual answer is, "Then don't use heterogeneous maps!" but...

## Case classes and similar aren't (always) the answer

Case classes are neat and tidy.  They specify exactly the fields to expect in their precise types.

Unfortunately, real-world data--particularly data ingested from another service--often isn't this tidy, and can change form over time in ways that are impossible to anticipate and unpredictable at exciting times.  This can be observed when a web service one consumes unpredictably changes its result format, requiring a cascade of changes to one's own case classes through an entire service in order to keep up.

Also document-oriented data--particularly if a document includes metadata--isn't always predictable enough to fit a schema defined by a set of Case Classes at all.  This is exacerbated if there is a requirement to generically process documents regardless of the metadata format, for example.  There simply may be more permutations than can be sensibly described precisely using case classes.

Anyone familiar with dynamic languages like Javascript or Clojure would immediately recognize that this is one problem ideally suited to using heterogeneous maps like the one illustrated above--but practice remains frowned-upon in the Scala community.

Where can we obtain insight about this?

First, I would like to suggest that we don't need to be dogmatic about our solution space here.  If another shoe fits better, let's wear it.  And I would agree that downcasting from `Any` is not in any way an ideal solution either.  So what should we do instead?

I think another layer in our application stack offers useful insights here.  Here's the story of that layer:

### Present-day Scala for applications ~=~ SQL for databases

A long time ago in a galaxy far far away, engineers from a number of mostly-large companies gathered in a darkened conference room to create a standard database language to assimilate all the others.

This new database language had to be as perfect as the smartest mathematicians and engineers could make it--which meant that it had to be perfectly precise, complete, and backed by mathematical proofs.  In the end their language became so perfect that it didn't allow a new row of data to be inserted unless every single field in its entity's schema was defined and every constraint satisfied.  This was a perfect guarantee indeed.

And for accounting applications--the prototypical application of that day--their solution proved to be about as perfect as the waterfall's product designers could require.

And if this isn't recognizable yet, I'm describing (with minor embellishment of course) relational databases generally and SQL specifically.

Which brings me to my first observation:  Strongly-typed languages share many of the same core values as SQL--along with similar advantages and disadvantages.

But what does this tell us about dynamic languages?  Let's continue the story...

### Dynamically-typed languages ~=~ Schemaless NoSQL databases

For all this mathematical perfection, every-day developers weren't always satisfied with their SQL solutions.

SQL databases could be brittle and hard to modify after they contained production data.  They required specialized training to perform all but the most trivial of tasks.  And worst of all, some kinds of data--particularly loosely-structured, hierarchical, or document-based data--couldn't be easily or efficiently stored and queried using them.

So engineers did what they do best--they invented new kinds of databases.  They affectionately described these new databases as NoSQL databases and importantly, many of these databases were and are "schemaless" by default.

And this bears a striking resemblance to the ways that programmers organize data using dynamic languages.

So, who won the database wars?  Actually, neither side "won" in the sense of gobbling up the majority of the other's market share.  But why?  And what can this teach us?

### Highly-structured vs. loosely-structured data

A key insight behind NoSQL database architectures when compared with SQL-based ones was that different kinds of data naturally have differing levels of structure.

It turns out that highly-structured data tends to fit relational databases well.  Loosely-structured data tends to fit databases with either no schema or a flexible schema well.  In other words, one could reasonably contend that:

* An entity is to SQL as a case class is to Scala
* A dynamic heterogeneous map in dynamic languages are to these languages as a dynamic heterogeneous map would be in Scala, **if there were an idiomatic way to express one.**

Per its name, Scala is a scalable language, designed to be extended from within.  Consequently, unlike with databases, an appropriate library ought to allow us to enjoy Scala's strong type safety when possible **and** utilize pragmatically-safe dynamic types at times when we may be required to process less-structured data.

How?  (I'm glad you asked!)

## Pragmatically-strong dynamic type safety

In order to solve this problem, let's talk about what features of dynamic languages can reduce downcasting risks and let's also list Scala features we would like to continue enjoying when applying dynamic types to Scala.

Here is a list of Scala values I feel are useful to carry forward:

* Immutability by default; mutable Vars only when explicitly specified
* Explicit types are better than downcasts
* Appropriate use of type inference improves readability

These dynamic language ideas could make sense in Scala

* A number is a number regardless of the internal representation it currently has.  For example, if I use an `Int` variable in a `Double` context, in practice there are few good reasons not to automatically promote the Int to a Double but the reverse conversion should be flagged.
* A date is a date is a date is a...  There are no good reasons that multiple timezone-specified Date classes shouldn't be assignment-compatible.
* By extension from the prior two, a string containing a number or a date can be used in place of a number or date

Here's how I believe we can put these together into a form that I believe provides a convenient way to process dynamic data (like JSON frequently is, for example).

### Each value must know its own type

Previously, our `Map` was of type `(key, value) = [Symbol, Any]`.  To allow the value part to be any type yet know its own type, we can create a boxed type that captures the value's type at initialization time.  One might think that adding a type parameter to a case class would do the job, but one would be wrong.  Here's why:

```scala
val pet = Map('name -> "Cocoa", 'breed -> "mutt", 'age -> 12)
// pet: Map[Symbol, Any] = Map('name -> "Cocoa", 'breed -> "mutt", 'age -> 12)
```

