# Classes and objects

Let's explore how classes and objects interact.  I'm assuming you're familiar with Java or C#'s object-oriented primitives.

First a regular class.

```scala
abstract class TimeLikeThings
```

```scala
class Date1(month: Int, day: Int, year: Int) extends TimeLikeThings

val d1 = new Date1(1, 10, 1968)
// d1: Date1 = repl.Session$App$Date1@2ab13644
val d1Prime = new Date1(1, 10, 1968)
// d1Prime: Date1 = repl.Session$App$Date1@5558bec2

d1 == d1Prime
// res0: Boolean = false
d1.hashCode
// res1: Int = 716256836
d1Prime.hashCode
// res2: Int = 1431879362
```

Let's add a naive implementation of these methods:

```scala
class Date2(month: Int, day: Int, year: Int) extends TimeLikeThings {
  override def hashCode = month + (12+day) + (12+31+year)
  override def equals(other: Any) = hashCode == other.asInstanceOf[Date2].hashCode
  override def toString = s"$month/$day/$year"
}

val d2 = new Date2(1, 10, 1968)
// d2: Date2 = 1/10/1968
val d2Prime = new Date2(1, 10, 1968)
// d2Prime: Date2 = 1/10/1968
val d2_ = new Date2(3, 10, 1970)
// d2_: Date2 = 3/10/1970

d2 == d2Prime
// res3: Boolean = true
d2_ == d2
// res4: Boolean = false
d2.hashCode
// res5: Int = 2034
d2_.hashCode
// res6: Int = 2038
```

And now we get the semantics one would expect.

Regular Scala class can be extended just like Java's classes, and again we get the results we expect.

```scala
class DateTime(month: Int, day: Int, year: Int, hours: Int, minutes: Int) extends Date2(month, day, year) {
    override def hashCode = super.hashCode + (3000 + hours) + (3000 + 24 + minutes)
    override def toString = s"$month/$day/$year $hours:$minutes"
}

val dt1 = new DateTime(1, 10, 1968, 4, 30)
// dt1: DateTime = 1/10/1968 4:30
val dt1Prime = new DateTime(1, 10, 1968, 4, 30)
// dt1Prime: DateTime = 1/10/1968 4:30
val dt2 = new DateTime(3, 10, 1970, 19, 17)
// dt2: DateTime = 3/10/1970 19:17

dt1 == dt1Prime
// res7: Boolean = true
dt2 == dt1
// res8: Boolean = false
dt1.hashCode
// res9: Int = 8092
dt2.hashCode
// res10: Int = 8098
```

## Things to try

* Make a Scala `Pet` class.  In addition to implementing `equals`, `hashCode`, and `toString`, add a `speak` method that returns an appropriate vocalization for your pet.

