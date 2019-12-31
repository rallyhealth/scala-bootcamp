### Prerequisites:

* [A class's companion](/bootcamp/snippets/classes-and-objects-pt2.md)
* [Classes and objects](/bootcamp/snippets/classes-and-objects-pt1.md)

--------

# Traits are like interfaces

Previously, we examined two of the main uses for singleton `object`s:

* As a distinct namespace to put things that belong together
* As "companion objects" for a class with the same name

In this snippet, we'll explore another important aspect of Scala's type system: traits and how they can function like dependency injection.

## Traits

A `trait` defines an abstract interface that is equivalent to modern Java's interfaces.

```scala
object Loggers {
    trait Logger {
        def error(t: Throwable, message: String): Unit = ???
        def warn(message: String): Unit = ???
        def info(message: String): Unit = ???
    }

    trait StdoutLogger extends Logger {
        override def error(t: Throwable, message: String) = {
            import java.io._

            val stringWriter = new StringWriter()
            t.printStackTrace(new PrintWriter(stringWriter))
            val stackTrace = stringWriter.toString.split("\n").take(7).mkString("", "\n", "\n at ....")

            println(s"[ERROR] $message\n$stackTrace")
        }

        override def warn(message: String) = println(s"[WARN] $message")

        override def info(message: String) = println(s"[INFO] $message")
    }

    // Let's write a meta-meta-logger!!!
    trait Slf4JLogger extends Logger {
        // Do slf4j things here...
        override def error(t: Throwable, message: String) = ???
        override def warn(message: String) = ???
        override def info(message: String) = ???
    }
}
```

We can inject this via normal inheritance:

```scala
object Animals {
    abstract class Animal extends Loggers.Logger {
        def logSpokenWord(word: String)
    }

    class HappyCat() extends Animal with Loggers.StdoutLogger {
        override def logSpokenWord(word: String) = info(s"purrrrr purrrrrrr $word pufffrrr")
    }

    class SleepyCat() extends Animal with Loggers.StdoutLogger {
        override def logSpokenWord(word: String) = warn("zzzzZZZZZzzzZZZ")
    }

    class GrumpyCat() extends Animal with Loggers.StdoutLogger {
        override def logSpokenWord(word: String) = error(new IllegalStateException(word),
            s"The worst part of my Monday....  is hearing you complain about your $word!")
    }

    val cuteKitty = new Animals.GrumpyCat()
}

Animals.cuteKitty.logSpokenWord("happiness")
// [ERROR] The worst part of my Monday....  is hearing you complain about your happiness!
// java.lang.IllegalStateException: happiness
// 	at repl.Session$App$Animals$GrumpyCat.logSpokenWord(classes-and-objects-pt3.md:58)
// 	at repl.Session$App$.<init>(classes-and-objects-pt3.md:66)
// 	at repl.Session$App$.<clinit>(classes-and-objects-pt3.md)
// 	at repl.Session$.app(classes-and-objects-pt3.md:3)
// 	at mdoc.internal.document.DocumentBuilder$$doc$.$anonfun$build$2(DocumentBuilder.scala:82)
// 	at scala.runtime.java8.JFunction0$mcV$sp.apply(JFunction0$mcV$sp.java:23)
//  at ....
```

Or through constructor injection, parameter injection, etc.

### But wait, there's more

We can also inject loggers at object construction time!

```scala
class BarkyDog() extends Animals.Animal {
  override def logSpokenWord(word: String) = warn("GgrrgrrrrrgggrraaAARRRfFF!")
}

val fido = new BarkyDog() with Loggers.StdoutLogger
// fido: BarkyDog with Loggers.StdoutLogger = repl.Session$App$$anon$1@67ce30c0

fido.logSpokenWord("Niiiiice doggy")
// [WARN] GgrrgrrrrrgggrraaAARRRfFF!
```

## Things to try

Because traits can contain implementation as well as interface/API they make it possible to separate concerns into single traits that are each mixed into a final whole.

Using the above example, separate `logSpokenWord` into a separate `Vocalization` trait hierarchy with various implementations appropriate to different kinds of animals.  Also create a `Locomotion` trait hierarchy.  Mix and match.

Play!  Have fun!
