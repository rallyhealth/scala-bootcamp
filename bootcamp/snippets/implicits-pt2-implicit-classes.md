### Prerequisites:

* [Implicit functions](/bootcamp/snippets/implicits-pt1-implicit-functions.md)

--------

# Implicit classes and open classes

We previously explained how implicit functions allow us to construct automatic type conversions.  This feature is often used to adapt an object to another type with a similar meaning as the first.  From this perspective, we created an implicit function to automatically adapt Scala functions to Java's `Runnable`.

Before we leave implicit functions there is one other less-common use that is worth exploring--that will also help us to motivate the next use for implicits we'll explore: implicit classes.

## Sometimes one type is another type's constructor input and...

Previously we illustrated how an implicit conversion of type `A => B` allows us to pass an instance of `A` as an argument to a method expecting a `B`.  Implicit conversion functions help us in another instance too.  Let's explain via illustration:

Consider how we typically construct and use a `java.io.File`:

```scala
def f1 {
  val f = new File(".")

  println( f.getCanonicalPath() )
}
f1
// /home/djo/code/scala-bootcamp
```

While the above code explicitly expresses **how** it works, sometimes we want terser code--that expresses its **intent** directly and cleanly while leaving out some of the implicit machinery.

In this example, if we declare an implicit conversion between `String` and `File`, we will obtain this effect.

```scala
def f2 {
  implicit def string2File(path: String): File = new File(path)

  println( ".".getCanonicalPath() )
  println( "/usr/share/dict/words".getCanonicalPath() )
}
f2
// /home/djo/code/scala-bootcamp
// /usr/share/dict/american-english
```

This is the other way to use implicit conversion functions.

> If we invoke a nonexistent method on an object, the compiler will look for an implicit conversion from the object's type to another type that **does** have the method.

Sometimes small casual Scala programs (particularly one-off programs) do things like this.

However, automatically converting String to File isn't great for general-purpose use for a different reason--it sometimes behaves in surprising ways:

```scala
def f3 {
  implicit def string2File(path: String): File = new File(path)

  println( ".".getCanonicalPath() )
  println( "/usr/share/dict/words".getCanonicalPath )
  println( "/usr/share/dict/words".exists )
}
f3
// error: type mismatch;
//  found   : String("/usr/share/dict/words")
//  required: ?{def exists: ?}
// Note that implicit conversions are not applicable because they are ambiguous:
//  both method augmentString in object Predef of type (x: String)scala.collection.immutable.StringOps
//  and method string2File of type (path: String)java.io.File
//  are possible conversion functions from String("/usr/share/dict/words") to ?{def exists: ?}
//   println( "/usr/share/dict/words".exists )
//            ^^^^^^^^^^^^^^^^^^^^^^^
// error: value exists is not a member of String
//   println( "/usr/share/dict/words".exists )
//            ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
```

In Scala, `String`'s `exists` method comes from an implicit conversion in Scala's standard library named `augmentString`.  Therefore, adding an additional implicit conversion between String and `File`, creates an ambiguity when both `File` and Scala's implicit conversion introduce matching method names.

If converting between `String` and `File` (or similar things) is a bad idea, when is this technique used?  Answer: When we write the object wrapper ourselves.

## Implicitly decorating for the holidays

Suppose we want to add `File`-like operations to String.  Converting directly to `File` isn't a great idea, but what if we wrote our own class containing just the needed `File` operations?

(This kind of wrapper class is sometimes called a [*Decorator*](https://dzone.com/articles/decorator-design-pattern-in-java) in design pattern literature)

```scala
def f4 {
    class FileOperationsDecorator(path: String) {
        import java.io.File
        private val f = new File(path)

        def fileExists: Boolean = f.exists              // #exists already works on String, so we have to rename
        def canonicalPath: String = f.getCanonicalPath  // more idiomatic Scala
    }

    implicit def string2FileOperations(path: String): FileOperationsDecorator = new FileOperationsDecorator(path)

    // Now Strings are automatically enhanced by methods in FileOperationsDecorator
    val path = "/usr/share/dict/words"
    (path.fileExists, path.canonicalPath)
}
f4
```

Using implicits in this way we can enhance any random class with methods its original author didn't implement, including classes in Java's standard library.

This leads to our next topics:

## Implicit classes

Enhancing someone else's class with additional methods is such a common idiom in Scala that it has its own name and syntax in the Scala language: an "implicit class".

Here's how it works:

In the example above, we created a custom class `FileOperationsDecorator` containing the extra methods we wanted to "add" to `String`.  Then we created a separate implicit function to convert from a `String` instance to our new `FileOperationsDecorator`.

Notice that `FileOperationsDecorator` only exists to add file operations to `String`.  It wouldn't make sense to construct one separately when the implicit conversion function does that for us automatically.

An implicit class lets us combine the implicit function syntax with class definition syntax.  **Effectively, implicit class syntax says that the class's primary constructor is its own implicit conversion function.**

```scala
object f5 { // Put implicits in their own scope
    implicit class FileOperationsDecorator(path: String) {
        import java.io.File
        private val f = new File(path)

        def fileExists: Boolean = f.exists              // #exists already works on String, so we have to rename
        def canonicalPath: String = f.getCanonicalPath  // more idiomatic Scala
    }

    // Now Strings are automatically enhanced to include methods in FileOperationsDecorator
    val path = "/usr/share/dict/words"
    (path.fileExists, path.canonicalPath)
}
f5
// res4: f5.type = repl.Session$App$f5$@c0c814c
```

## Next time

We've examined implicit type conversion functions and implicit classes.  Next time we'll illustrate the techniques we've seen so far using a real-world case study.

Then we'll begin exploring the third kind of implicit in Scala: implicit values and implicit parameters.

## Things to try

Implicit functions and implicit classes are frequently used together to extend Scala's syntax or to create domain-specific languages.  Here are two smaller examples you can try:

* `java.io.File` is nice, but we can make it nicer still.  Use an implicit class to add a directory separator operator to `File`.  For example, one might like to be able to write: `new File("/")/"etc"/"passwd"` to obtain a `File` object pointing to `/etc/passwd`

* Extend your implicit class from the prior example to add `.slurp(): String` and `.spit(content: String): Unit` to `java.io.File` for reading/writing text files.
