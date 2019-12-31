### Prerequisites:

* [Classes and objects](/bootcamp/snippets/classes-and-objects-pt1.md)

--------

# A class's companion

[Last time](classes-and-objects-pt1.md) in this series we illustrated how "regular" Scala classes behave just like Java classes, but with different syntax:

* Abstract classes
* Classes don't provide default implementations of `equals`, `hashCode`, or `toString`.
* Inheritence works as one would expect.

Here I'm going to talk about `object`s and how they often work together with `class`es as what are known as "companion objects".

## Object

An `object` is a singleton.

```scala
object Date {
    // methods here can only exist on the single Date instance
    def hello = println("I only exist on the singleton Date object")
}

// Invoke Date.hello
Date.hello
// I only exist on the singleton Date object
```

Objects and classes live in two separate namespaces.  This means that you can define a class with the same name as another object.  Here's the `Date` class from last time:

```scala
class Date(month: Int, day: Int, year: Int) {
    override def hashCode = month + (12+day) + (12+31+year)
    override def equals(other: Any) = hashCode == other.asInstanceOf[Date].hashCode
    override def toString = s"$month/$day/$year"
}

val d1 = new Date(1, 10, 1968)
// d1: Date = 1/10/1968
Date.hello
// I only exist on the singleton Date object
```

By examining the `new Date` and the `Date.hello` calls, we can see that both the `Date` class and the `Date` object are coexisting.  But what's this warning about "companions"?

It turns out that defining a class and a corresponding object with the same name is a really common thing to do in Scala when the two have a symbiotic relationship.  What does that mean?  Let's look at one example.

To define a companion `class`/`object`, we just have to define both the class and companion object in the same scope with each other.  And to create that single scope, we'll use an `object` as it's own namespace--another common use for `object` in Scala.  (See what I did there? 😁)

```scala
object DateStuff {

    object Date {
        def apply(month: Int, day: Int, year: Int): Date = new Date(month, day, year)
    }

    class Date(month: Int, day: Int, year: Int) {
        override def hashCode = month + (12+day) + (12+31+year)
        override def equals(other: Any) = hashCode == other.asInstanceOf[Date].hashCode
        override def toString = s"$month/$day/$year"
    }

    val aDate = Date(3, 10, 1970)  // Look Ma, don't need the `new` operator!
}

DateStuff.aDate
// res2: DateStuff.Date = 3/10/1970
```

Have we seen this somewhere else before in Scala?  Probably.  If you've ever created a List, you've already used a companion object; it's why you can omit the `new` operator when constructing a List...

```scala
List(1, 2, 3)
// res3: List[Int] = List(1, 2, 3)
```

Previously in this series we saw how regular classes and abstract classes obey the rules one would expect if a person has a background in Java or C#.

Here we discussed how Scala has special syntax for singleton objects and that these kinds of objects are often used as "companions" to classes with the same name.  I also snuck in an example of `object` being used as its own namespace.


## Objects and domain models

Since a `class` and an `object` together define their own namespace, a common Scala idiom is to separate the object's identity from its behavior.  This means:

* A `class` only specifies the class's fields, along with `equals`, `hashCode`, and `toString`.
* Other methods live on the class's companion object, accepting the class instance as an initial parameter.  For example:

```scala
class Pet()

object Pet {
  def apply(): Pet = new Pet
  def runHome(self: Pet): Unit
}
```

For this trivial example there is no benefit to expressing things this way, but as pets (and the things they do) become more and more complicated, this approach provides more flexibility over placing all behavior inside a model class.

## Something to try

Update your `Pet` class from last time to feature a companion object that implements all of its behavior as described here.
