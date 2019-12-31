# Introduction

```scala
{
  val fable =
    ("Nouns *AND* verbs, what a thought!" ::
    "I'd heard they'd fought" ::
    "War!" ::
    "Internet said, 'WOT?  NOT!'" ::
    "Verbs are still an afterthought?" ::
    "Noire..." ::
    "A val, he kills" ::
    Nil)

  fable(0)
}
// res0: String = "Nouns *AND* verbs, what a thought!"
```

Coming from Java, programmers are used to (nearly) everything being a noun (or a verb attached to one).
But Scala gives us additional options.

This article's purpose is not to be opinionated about functions versus objects as much as to
explain and to illustrate an object/functional idiom equivalent to a single object-oriented idiom
Java programmers are used to.

# Domain objects

Java programmers will be familiar with the following way to model a domain object:

```java
import java.io.PrintWriter;
import java.io.OutputStream;

public class Person {
  private String firstName;
  private String lastName;

  public Person(String firstName, String lastName) {
    this.firstName = firstName;
    this.lastName = lastName;
  }

  // Getters/setters...
  public String getFirstName() {
    return firstName;
  }

  public void setFirstName(String firstName) {
    this.firstName = firstName;
  }

  public String getLastName() {
    return lastName;
  }

  public void setLastName(String lastName) {
    this.lastName = lastName;
  }

  // Some methods...
  public String greet(String greeting) {
    return greeting + " " + firstName + " " + lastName;
  }

  public void printGreeting(OutputStream out, String greeting) {
    PrintWriter w = new PrinterWriter(out);
    w.print(greet(greeting));
    w.flush();
  }
}
```

Why would it be useful to begin with an example so elementary that any experienced Java developer would take it for granted?

Let's consider:

* The Java domain object above uses mutable instance members.  This is the expected style in Java, but not the preferred approach in idiomatic Scala.
* Scala's notion of object-oriented programming instead encourages creating objects that cannot be changed once created.

Scala's object-oriented style (expressed using Java) might look something like:

```java
import java.io.PrintWriter;
import java.io.OutputStream;

public class Person {
  public final String firstName;
  public final String lastName;

  public Person(String firstName, String lastName) {
    this.firstName = firstName;
    this.lastName = lastName;
  }

  public String greet(String greeting) {
    return greeting + " " + firstName + " " + lastName;
  }

  public void printGreeting(OutputStream out, String greeting) {
    PrintWriter w = new PrinterWriter(out);
    w.print(greet(greeting));
    w.flush();
  }
}
```

Or when translated directly into Scala:

```scala
import java.io.PrintWriter
import java.io.OutputStream

class PersonModel(firstName: String, lastName: String) {

  def greet(greeting: String) = greeting + " " + firstName + " " + lastName

  def printGreeting(out: OutputStream, greeting: String) : Unit = {
    val w = new PrintWriter(out)
    w.print(greet(greeting))
    w.flush
  }
}
```

# Singularly nouns _and_ verbs

Let's pause here to see what we have done so far.

Notice that at its essence, Scala's object-oriented style defines a group of functions (methods) that
apply to constant values (the object) along with their own additonal function parameters.

But consider this:

Since the object instance is just a bunch of constants anyway, one might separate the class declaration from
the functions (methods) that operate on instances of that class.  For example:

```scala
case class Person(firstName: String, lastName: String)

object Greeter {
  def greet(p: Person, greeting: String) = greeting + " " + p.firstName + " " + p.lastName

  def printGreeting(p: Person, out: OutputStream, greeting: String) : Unit = {
    val w = new PrintWriter(out)
    w.print(greet(p, greeting))
    w.flush
  }
}
```

But why would we want to?  For fun, let's compare and contrast the original OO approach with this
object/functional approach:

## OOOOhhhh

Unsurprisingly, here's how one would construct and use the OO domain model.

```scala
val porkey = new PersonModel("Porky", "Pig")
// porkey: PersonModel = repl.Session$App$PersonModel@ece8659
porkey.greet("Hello")
// res1: String = "Hello Porky Pig"
porkey.greet("Hola")
// res2: String = "Hola Porky Pig"
```

## Functionally functional

A naive approach to writing this same code using a functional approach might look like the following:

```scala
val mickey = Person("Mickey", "Mouse")
// mickey: Person = Person("Mickey", "Mouse")
greet(mickey, "Hello")
// res3: String = "Hello Mickey Mouse"
greet(mickey, "Hola")
// res4: String = "Hola Mickey Mouse"
```

The object-oriented example above creates an instance of our PersonModel
class and calls its `greet` method using standard `instance.method()` syntax.

In the object-functional example, we create an instance of the `Person` case class, and then call the
`greet` function, passing the `Person` instance as the initial parameter.

### But why...?

So far, this example is about as unsurprising as it can get.  There are hardly any differences between
the object-oriented calls and the object-functional ones, and the object-functional Person object
required slightly more ceremony to set up to begin with.

Therefore, a reasonable question at this point would be, "Does the functional approach have benefits,
even with this simple example?"

Yes.

Where are they?

The remainder of this article will examine just one benefit and will discuss some of its implicaitons.

# Partial function application

Let's begin by considering another way Scala allows us to call the `greet` function:

```scala
val hiMickey = greet(mickey, _: String)
// hiMickey: String => String = <function1>
val yoDude = greet(_: Person, "Yo")
// yoDude: Person => String = <function1>
```

## $$$Lambda$-what?

Yes:  calling our `greet` function this way--by passing wildcards in place of parameter values--
does not actually call `greet`, but returns a new lambda (anonymous) function instead.

Further, as the method signatures in the REPL indicate, these new functions accept a *single* parameter--
the paramter where we passed the wildcard--and continue to return the result of the original
`greet` function.

In other words, by using wildcards, we can redefine `greet` to hold *either* the `Person` parameter
*or* the `greeting` parameter constant, returning a new function mapping the remaining parameter
to the resulting greeting.

## How do we use this?

In the `OO` idiom, we instantiate an object (to which methods are preassigned).  The identity of
the object is specified by the values passed to its constructor arguments.  Then we can call functions,
or methods, that are preassigned to that object's class, using the object instance as the subject
of the sentence.  For example:

* `mickey.greet("Bonjour")`

To use this object-functional idiom, rather than instantiating an object (to which methods are
preassigned), we define a Scala function accepting all arguments required to greet something.

Then we call this function without supplying values for all argument list parameters, but passing
wildcards for unspecified parameter values.  The result is a new function mapping the wildcard
parameters to the function's result type and holding the specified parameter values constant.

We have now achieved a result similar to instantiating a class containing a single method:
the parameters that were assigned values are now held constant, similar to constructor parameters.
And the new function we just created can be utilized in the exact same manner as a method call
on an object instance, but with less syntactic overhead.

For example, instead of `mickey.greet("Hello")` we can now write:

```scala
hiMickey("Hello")
// res5: String = "Hello Mickey Mouse"
hiMickey("Hola")
// res6: String = "Hola Mickey Mouse"
```

And instead of something like `yoGreeter.greet(mickey)`, we can write:

```scala
yoDude(mickey)
// res7: String = "Yo Mickey Mouse"
yoDude(Person("Donald", "Duck"))
// res8: String = "Yo Donald Duck"
```

It's worth noting again both `hiMickey` and `yoDude` were derived by **extending** the **same**
`greet` function.

This technique--calling a function with an incomplete argument list in order to obtain a new
function with the defined parameters held to constant values--is called *partial function application.*


## Flexibility, schlepsibility...

The reader might wonder at this point, "I can see how having this kind of flexibility can be useful
in theory, but this particular example seems fairly contrived; can we do better?"

While admittedly most programmers don't consider generating a greeting to be their killer app,
we do generate other kinds of data all the time.  And frequently, the destination for this
data may vary depending on if it is being logged, being forwarded to a network connection,
being printed on the screen, and so forth.

So by analogy, let's consider a case where an arbitrary `Person`s' greetings must
be delivered to an arbitrary destination, where the destination is the thing that will be defined
early in the program and held constant.

Using the design we have described so far, a programmer can extend the `printGreeting`
function to send arbitrary greetings to a constant output channel.  (For purposes of illustration,
we'll use an OutputStream here.)

Recall the code we introduced above:

```scala
object Greeter {
  def greet(p: Person, greeting: String) = greeting + " " + p.firstName + " " + p.lastName

  def printGreeting(p: Person, out: OutputStream, greeting: String) : Unit = {
    val w = new PrintWriter(out)
    w.print(greet(p, greeting))
    w.flush
  }
}
```

Let's capture all output in a `ByteArrayOutputStream`:

```scala
val networkPayload = new ByteArrayOutputStream()
// networkPayload: ByteArrayOutputStream = Howdy Mickey Mouse
val sendableGreeting = printGreeting(_: Person, networkPayload, _: String)
sendableGreeting(mickey, "Howdy")
networkPayload.toByteArray
// res10: Array[Byte] = Array(
//   72,
//   111,
//   119,
//   100,
//   121,
//   32,
//   77,
//   105,
//   99,
//   107,
//   101,
//   121,
//   32,
//   77,
//   111,
//   117,
//   115,
//   101
// )
```

Now we can greet anyone anywhere!

And suppose the story our program is telling is about Mickey Mouse.  We can trivially extend this
function to build a `mickeySerializingGreeter` that holds both the OutputStream and the Person
parameters constant:

```scala
val comicStripSender = new ByteArrayOutputStream()
// comicStripSender: ByteArrayOutputStream = What's up, Doc, Mickey Mouse
val mickeySerializingGreeter = printGreeting(mickey, comicStripSender, _: String)
mickeySerializingGreeter("What's up, Doc,")
comicStripSender.toByteArray
// res12: Array[Byte] = Array(
//   87,
//   104,
//   97,
//   116,
//   39,
//   115,
//   32,
//   117,
//   112,
//   44,
//   32,
//   68,
//   111,
//   99,
//   44,
//   32,
//   77,
//   105,
//   99,
//   107,
//   101,
//   121,
//   32,
//   77,
//   111,
//   117,
//   115,
//   101
// )
```

# Making objects on the fly

When using partial function application to create new functions from existing ones, we are free to
use **any group** of function parameters--not just the initial one--as the subject or the `this` object of
the sentence.

The second example created a `Greeter` object from the same initial function.  This object greets
any kind of `Person` object using the greeting specified initially.

Said differently, the functional approach allows `greet` to be used as if it were a member of the
Person class or a member of a Greeting "class".

"Wait!  We never defined a Greeting class, we just used a String to contain our greeting!" one
might reply.

Exactly!

Using this technique, there is no need for a specialized Greeting class.  Partial function
application allows us to treat *any* of the `greet` function's parameters as a the subject of a
sentence--as the object to which the "method" belongs.

Further, creating a specialized Greeting class that is able to greet various Person objects
would only add boilerplate and would lead to a dillemma--to what class should `greet` belong?
Or should it be *duplicated* across both classes?

The advantage of choosing the object-functional approach in this kind of situation is that the programmer
does not need to decide in advance to what class a method must belong.  A `Person` can issue arbitrary
greetings, *and* a `Greeter` may greet arbitrary `Person`s--using the same code.

And as in the last example, both a `Person` and an `OutputStream` can be bundled together as if they
were mixed-in to the same `class`--on the fly.

## When would a person avoid this technique?

The primary occasion I can imagine avoiding this technique altogether is when a class has more than a handful
of methods.  In that situation, the boilerplate involved in creating each "method" on the fly
via partial function application may not be worth the cost.

# Noire... / A val, he kills / Nil

Hopefully, some uses for this technique are starting to become apparent.

It's not the only hammer.  Sometimes classes and traits are the right solution.

That said, I hope that if you haven't considered partial function application  before, that this discussion
has made it easier to see how and where it might be a useful addition to your toolbox.
