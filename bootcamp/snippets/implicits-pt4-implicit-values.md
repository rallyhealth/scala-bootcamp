### Prerequisites:

* [Traits are like interfaces](/bootcamp/snippets/classes-and-objects-pt3.md)

--------

# Implicit values and parameter dependency injection

Today we're going to introduce implicit values, Scala's final use for the `implicit` keyword, and illustrate one way to use them via parameter dependency injection.

Like the other snippets we'll start by explaining our topic: implicit `val` and implicit parameters.  However, like other fairly advanced Scala topics, the application may not be immediately obvious.  So we'll conclude with a worked example showing one way you can use this feature to simplify your code right away.

Consequently, in a sense, this is really a "Two Snippets for the Price of One" deal!  Enjoy!

Oh, and that's not all!

I'm planning for the next snippet (or few) to round out our series on implicits by introducing one of the most important and common patterns in Scala: the Typeclass pattern.

Have fun, and see you again soon!

## The two faces 🤔 of Scala's implicit values 🙂<sup>[1](#Footnotes)</sup>

Scala has two kinds of implicit values: *implicit vals* and *implicit parameter lists*.  These correspond with producers of implicit values and consumers of implicit values.

### Implicit vals

Here's how you define an `implicit val`.

```scala
trait Pet {
    val name: String
}

case class Dog(override val name: String, barkVolumeDb: Double) extends Pet

implicit val myDog = Dog("Cocoa", 120)  // 🤩
// myDog: Dog = Dog("Cocoa", 120.0)  // 🤩
println(myDog)
// Dog(Cocoa,120.0)
```

So far, our implicit val is just acting like a regular `val`.  That's because we still need to define an
implicit parameter list to consume it.

### Implicit parameter lists

Implicit parameter lists are declared similarly to implicit vals: by adding the `implicit` keyword.

```scala
def printPetName(implicit pet: Pet) = println(pet.name)  // 🤠  // 🤠
printPetName(myDog)
// Cocoa
```

Our implicit parameter can act like a regular parameter just like the implicit val.  What else *can* we do with these that we couldn't do with regular vals or parameters?

## Parameter dependency injection

Implicit values function as a kind of parameter dependency injection.  Here's how:

### What *do* implicit values do differently?

```scala
printPetName   // myDog was injected automatically because she's the only Pet in scope!  🤯
// Cocoa
```

In plain English, any parameter in an argument list with `implicit` at the front will automatically be filled with any type-compatible implicit val that is in scope at the time the function is invoked, as long as there is only a single one in scope.<sup>[2](#Footnotes)</sup>

This just means that when we called `printPetName` above, we didn't have to pass any parameter at all!

Because `myDog` is already an implicit value, it's in scope, and it's the only implicit value matching the `printPetName` parameter type.  Consequently, Scala's compiler will automatically pass `myDog` if we don't pass something else ourselves.

If more than one implicit Pet is available in the scope of a call to `printPetName`, Scala's compiler will automatically flag this as an error.

### Implicitly injecting

Okay, what can we use this for?

The comment after `printPetName` above already hints at the answer.

If we squint hard enough, this behavior looks a lot like parameter dependency injection--just on a micro scale.  Could this also work at application scale?

Yes.

### Modular dependency injection modules

Here's an example where we inject a DatabaseModule into an Application:

> **Note:** By far this isn't the only way to implement dependency injection in Scala applications, I suggest asking your team what is the team's usual way to do this before using this technique in your own codebase.

```scala
trait DatabaseConnection {
    def ask(query: String): String
}

class FakeDatabase extends DatabaseConnection {
    override def ask(query: String) = query + ": fake results"
}

class AlexaOnMyDesk(connectionString: String) extends DatabaseConnection {
    override def ask(query: String) = query + ":\n>>>> Error: 🤮\n>>>> Error: Customers not found.\n>>>> Cause: Those people didn't sit still.\n>>>> Cause: They all moved around starting Web sites, one after another after ano..."
}

class DatabaseModule(environment: String, etcetcetcetera: String) {
    val connection = environment match {
        case "scalatest" => new FakeDatabase
        case "laptop" => new AlexaOnMyDesk("get connection string from etcetcetcetera")
        // ...
    }
}

class CustomerService {
    def getAll(implicit db: DatabaseModule): String = {
        db.connection.ask("Hey Alexa: Find me all the customers from before the .com crash")
    }
}

class Application(implicit db: DatabaseModule) {
    val customers = new CustomerService

    def partyLikeIts1999 = println(customers.getAll)
}

implicit val database = new DatabaseModule("laptop", "etcetcetcetera")
// database: DatabaseModule = repl.Session$App$DatabaseModule@19d11d8b

val app = new Application   // The current DatabaseModule is automatically injected here
// app: Application = repl.Session$App$Application@fac1a8b   // The current DatabaseModule is automatically injected here

app.partyLikeIts1999
// Hey Alexa: Find me all the customers from before the .com crash:
// >>>> Error: 🤮
// >>>> Error: Customers not found.
// >>>> Cause: Those people didn't sit still.
// >>>> Cause: They all moved around starting Web sites, one after another after ano...
```

When running on a developer laptop the application will configure one database connection in the database module.  When running tests, the application will configure a different one.  When running in each sever-side environment, yet others.

But due to parameter dependency injection the application will automatically use the correct database connection for its environment.

## Conclusion

In this snippet we have illustrated how implicit values collectively work together to enable a form of parameter dependency injection that is directly supported by Scala.

Beyond implementing dependency injection, implicit values are used to implement Type Classes, or the Typeclass pattern.  Type classes are a way to implement extra-flexible polymorphism without inheritence and are used all over Scala's standard libraries.

## Things to try

> **Note**:
> 
> This chapter's "things to try" is more challenging than other chapters.  It's worth it!  We promise!  It'll make the next chapter easier to understand.  And feel free to get help from your teammates...

In the example above, suppose we add a `DatabaseConnectionFactory` type-parameterized to the current environment--as follows:

```scala
object DatabaseConfig {
  trait Environment
  class TestEnv extends Environment
  class ProdEnv extends Environment

  trait DB[E <: Environment] {
    def ask(q: String): String
  }

  class TestDb[E <: Environment] extends DB[E] {
    override def ask(query: String) = query + ": fake results"
  }

  class Hal[E <: Environment] extends DB[E] {
    override def ask(query: String) = "I'm sorry Dave, I just can't do that."
  }

  implicit val stub = new TestDb[TestEnv]
  implicit val hal = new Hal[ProdEnv]
}
```

Notice that in this instance two implicit database vals are in scope: both `stub` and `hal`.  How might we write a query method so that depending on the current environment in scope, the correct database will automatically be chosen?  Modify the following example to make it behave as expected:

```scala
// Given:
import DatabaseConfig._
implicit val env = new ProdEnv

// Change this line so that
def query(q: String): String = db.ask(q)

// this line automatically picks up the implicit Hal instance
query("Open the cargo-bay doors")
```

#### Footnotes

1. Okay, I oversimplified a little.  There are a couple of other kinds of implicit "vals".  To name a few: `implicit def name: type = <expression>`, is an implicit value that is recalculated every time it's used.  `implicit lazy val name: type = <expression>` is an implicit value that is calculated the initial time it's used.  But at the end of the day these are just other ways to make implicit values to be consumed by implicit parameters.
2. Implicit scope is more complicated than this.  But for this article's purpose this oversimplified definition is good enough.

