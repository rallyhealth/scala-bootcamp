### Prerequisites:

* [A class's companion](/bootcamp/snippets/classes-and-objects-pt2.md)
* [Classes and objects](/bootcamp/snippets/classes-and-objects-pt1.md)
* [Traits are like interfaces](/bootcamp/snippets/classes-and-objects-pt3.md)

--------

# Classes and objects, part 4

Today we'll have fun exploring that other kind of class Scala supports: `case class`es.

This snippet builds on prior topics in this series.  Here is a brief summary/recap of these topics:

* Classes, abstract classes, equals, hashCode, and toString
* Objects as standalone namespaces; objects as "companions" to regular classes and the special relationship these enjoy
* Traits: equivalent to modern Java's interfaces; usefulness for injecting behavior into classes

## Case classes

Case classes are usually used as domain model classes.<sup>[1](#Endnotes)</sup>

```scala
object Animals {
  case class Dog(name: String, breed: String, barkFrequencyPeaksHz : Set[Int])
  case class Cat(name: String, breed: String, meowFrequencyMillis: Int, sleepFrequencyMillis: Int)
  case class Parrot(name: String, favoritePhrases: Set[String])
}
```

One reason case classes are so handy for domain modeling is that Scala automatically generates sensible `equals`, `hashCode`, and `toString` methods for them.  It also generates a companion object for the class with an `apply` method so the class name can be its own constructor.  For example, given:

```scala
object Example1 {
  import Animals._

  val dog1 = Dog("Cocoa", "Terrier", Set(1500, 2000, 3000, 6000))
  val dog2 = Dog.apply("Cocoa", "Terrier", Set(1500, 2000, 3000, 6000))
}
```

```scala
import Example1._
```

The following will work as expected:

```scala
dog1 == dog2
// res0: Boolean = true
dog1.hashCode == dog2.hashCode
// res1: Boolean = true
dog1.toString
// res2: String = "Dog(Cocoa,Terrier,Set(1500, 2000, 3000, 6000))"
```

In addition, Scala generates an `unapply` method in the companion object that can be used to retrieve the case class's original arguments.  This is how Scala unpacks case classes in `match`/`case` statements.

```scala
object Example2 {
  import Animals._

  Dog.unapply(dog1)
}
```

The thing inside the Option is a Tuple2[String,String].  Depending on the source type and the desired outcome, `unapply` can return other things too  (Google can tell you more).

## Case class limitations

With all these cool features, why wouldn't we want to use case classes everywhere?  The biggest limitation is that a case class cannot inherit from / `extend` another case class.  So, if we define a `case class Species`, Scala will yell if we try to extend it.

```scala
object Example3 {
  case class Species(speciesName: String)
  case class Dog(name: String, speciesName: String, breed: String, barkFequencyPeaksHz: Set[Int]) extends Species(speciesName)
}
// error: case class Dog has case ancestor repl.Session.App.Example3.Species, but case-to-case inheritance is prohibited. To overcome this limitation, use extractors to pattern match on non-leaf nodes.
//   case class Dog(name: String, speciesName: String, breed: String, barkFequencyPeaksHz: Set[Int]) extends Species(speciesName)
//   ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
```

Contrary to the error message above, the usual workaround is to use traits.

```scala
object Example4 {
  trait Animal {
    val speciesName: String
  }

  case class Dog(name: String, breed: String, barkFrequencyPeaksHz : Set[Int]) extends Animal {
    override val speciesName="Canis Familiaris"
  }

  case class Cat(name: String, breed: String, meowFrequencyMillis: Int, sleepFrequencyMillis: Int)   extends Animal {
    override val speciesName="Felis catus"
  }

  case class Parrot(name: String, favoritePhrases: Set[String]) extends Animal {
    override val speciesName="Psittaciformes"
  }
}
```

We can use these case classes as one would expect, including pattern matching on them.  But there's one more cool trick available.

## Closed sets, case classes, and sealed traits...

Sometimes it's useful to model a parent trait and its child case classes as a closed set.  In this situation, Scala's compiler will check all of our `match`/`case` statements and issue a warning if a particular `match` block is not exhaustive.  This can be a lifesaver when one realizes a need to add a new `case class` to a trait and then has to hunt down all of the `match` statements involving that trait.  Here's an example:

```scala
object Animalia {
    sealed trait Animal {
        val speciesName: String
    }

    case class Dog(name: String, breed: String, barkFrequencyPeaksHz : Set[Int]) extends Animal { override val speciesName="Canis Familiaris" }

    case class Cat(name: String, breed: String, meowFrequencyMillis: Int, sleepFrequencyMillis: Int) extends Animal { override val speciesName="Felis catus" }

    case class Parrot(name: String, favoritePhrases: Set[String]) extends Animal { override val speciesName="Psittaciformes" }

    def getBreed(a: Animal): Option[String] = a match {
      case Dog(_, breed, _) => Some(breed)
      case p: Parrot => None
    }
// warning: classes-and-objects-pt4.md:136:47: match may not be exhaustive.
// It would fail on the following input: Cat(_, _, _, _)
//     def getBreed(a: Animal): Option[String] = a match {
}
```


Go Forth And Domain Model!

### Endnotes

1. [Bark frequencies](http://article.sciencepublishinggroup.com/pdf/10.11648.j.ijmea.20140201.14.pdf)

