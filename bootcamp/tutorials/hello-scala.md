# Hello, Scala! From installation to first run

This a fork/update of Daniel Spiewak's [Getting Started in Scala](https://gist.github.com/djspiewak/cb72c41ac335a3a9b28b3307be04aa43) with minor tweaks for Rally's environment.

# Getting Started in Scala

This is my attempt to give Scala newcomers a quick-and-easy rundown to the prerequisite steps they need to a) try Scala, and b) get a standard project up and running on their machine. I'm not going to talk about the language at all; there are plenty of better resources a google search away. This is just focused on the prerequisite tooling and machine setup. I will _not_ be assuming you have any background in JVM languages. So if you're coming from Python, Ruby, JavaScript, Haskell, or anywhere... I hope to present the information you need without assuming anything.

One assumption I'm _knowingly_ making is that you're on a Unix-like platform or using a Linux environment inside of Windows (WSL 2 or later, Hyper-V VM, etc...)

## Install Homebrew if you haven't already

*(if you're on a Mac)*

[Homebrew home page](http://brew.sh/)

## Getting the JVM

Since many Scala tools (particularly IntelliJ) remain pinned to Java 8, that is the version you should install.  You can [install Oracle's JDK](https://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html) or [AdoptOpenJDK's version](https://adoptopenjdk.net/).

Here's a simple recipe for obtaining *AdoptOpenJDK* on a Mac:

```bash
$ brew tap AdoptOpenJDK/openjdk

# ...lots of stuff scrolls by, then

$ brew cask install adoptopenjdk8
```

On Linuxes, *OpenJDK* 8 is generally already pre-packaged; just search your distribution's package manager.

Now it's time to tool up for Scala.

## Install IntelliJ

Most Rally engineers use IntelliJ as their main Scala IDE.  If you haven't already, go to the [IntelliJ web site](https://www.jetbrains.com/idea/) and download Idea Ultimate.  There's a 30 day trial period, and you can get a full license by filing a ticket with IT.

Its main advantages as of this writing are integrated refactoring and integrated debugger.  Once mastered, refactoring can be a coder's superpower...  The first time you run it, remember to install the Scala plugin and you'll be all set!

## Or install VSCode with the Metals plugin for Scala

If you're coming from a Microsoft background you might find this more comfortable.

As of this writing, VSCode with Metals can search across open projects, has a similar interactive coding experience as IntelliJ, and consistently reports the same error messages in the editor as the build tool does from the command line.

## Trying Scala

At the present time, there is no _great_ solution for trying out Scala in your browser without installing anything. There are a few implementations of the [Scala Fiddle](https://scalafiddle.io/) concept, and the one linked is (I think) the best. Because of technical limitations at present, the turnaround time is longer than one would like, and it's difficult to play around with libraries.

I'm going to recommend that instead you go to http://ammonite.io and install the Ammonite REPL there.

Ammonite provides a Scala REPL with syntax coloring and reasonable error messages. _For the most part_, any Scala tutorial snippets you find on the internet should work in this environment. The exceptions are related to overloading, scoping, and class companions. But if you aren't doing anything related to those things, you should be fine.

Go ahead and install it now; I'll wait...

## Here we go!

Here's a quick try-out:

```scala
@ println( "Hello, World!" )
Hello, World!
```

To exit the REPL, type `exit`. Ctrl-C will kill a long-running expression (e.g. if you accidentally write an infinite loop). To run it again, type `amm` again.

The Ammonite REPL is generally better than the stock Scala REPL. I make use of it on a daily basis! It's worth reading the documentation over at `ammonite.io` to see what else you can do with it.

## Install SBT

`SBT` is Scala's standard build tool, much like `make`, `maven`, and others for other languages and it's what we use at Rally.  On a Mac you can get it via:

```bash
$ brew install sbt
```

SBT can sometimes reach MacOS's maximum file handle limit. Fix this by adding the following to ~/.profile (or to ~/.bash_profile or ~/.zshrc, if that's how you roll):

```bash
$ ulimit -S -n 4096
```

Similarly, you may experience various out of memory errors using sbt. Like above, fix by adding to your login script:

```bash
export SBT_OPTS="-XX:MaxMetaspaceSize=1g -Xmx2024m -Xss2m"
```

Then reload your bash settings:

```bash
$ source ~/.profile
```

At Rally we currently have both SBT version 0.13.x and SBT 1.x.  If you have issues running sbt in a project that is using sbt 0.13.x, you may need to install an older version of sbt via brew:

```bash
$ brew install sbt@0.13
```

Then you will need to set your sbt version in `/usr/local/etc/sbtopts`.  Look for the following line and uncomment and use a 0.13.x version of sbt:

```bash
# Sets the SBT version to use.
-sbt-version 0.13.17
```

## Create a Project

I would love it if I could say at this point "just run this one command and you're done!" But sadly, there's some more boilerplating we need to do.

The first step is to create your project's directory structure. I generally do something like this (for a project named funzles):

```bash
$ mkdir -p funzles/src/{main,test}/scala/funzles
$ cd funzles
```

This is generally followed by a `git init`, but we'll leave that for now. The `src/main/scala/` directory convention comes from Maven, a build system and package manager for Java. It does have some advantages, but you're certainly going to have to get used to some very deeply nested directories! Generally speaking, you'll put your main project source files under `src/main/scala/`, while your tests are going to go in `src/test/scala/`. It is possible to organize your project files differently, but SBT understands this convention right out of the box, and other Scala developers will be very at home in your project if you stick to the standards.

### Configuring Your Build

Now we need to configure SBT so it understands more about our project than the fact that we're using Scala. To do this, we need to create a file named `build.sbt` in the root directory of your project. For now, the following contents should be sufficient:

```
name := "funzles"

scalaVersion := "2.11.12"
```

> As of this writing, `2.11.12` is the latest Scala version in use in Rally Connect (with `2.12`in the pipeline). In production Rally projects you won't have to worry about the current Scala version because we specify this using a standard SBT plugin containing settings that are common across projects.

A brief aside about SBT... SBT's configuration syntax (in `.sbt` files) is very extensive. There's a _ton_ of stuff you can do with it. I do not recommend falling down that particular rabbit hole if you're new to Scala! If you decide to start doing more complex things, there are resources to help you do more with SBT (I recommend the book *[SBT In Action](https://www.manning.com/books/sbt-in-action)* as well as the [official SBT documentation pages](http://www.scala-sbt.org/0.13/docs/index.html)).

### Building and Running

Fortunately, we're all done with the boilerplate stuff! We can actually start doing real coding with a real project (sort of). Start by launching SBT (assuming you put the `sbt` script from earlier on your `PATH`):

```bash
$ sbt
```

In general, you're going to want to have SBT open in your project whenever you're doing development work. As you probably could see from running that command, SBT takes a bit of time to start (though much longer when it's your first time!). The SBT prompt offers several commands which are extremely useful, and keeping SBT itself open allows it to cache results and be significantly more efficient, reducing turnaround time on compilation and testing and increasing your productivity. Some quick commands to try:

- `compile` -- Compiles all of the _main_ source files in your project. This task is incremental: if a file hasn't changed, and its dependent files have not changed, it will not be recompiled.
- `test` -- Compiles (if necessary) and runs all of the _test_ source files in your project. Main source files will also be compiled if necessary.
  - `testQuick` -- Compiles your main and test files (if necessary) and calculates which tests (relative to your last test run) have been affected by file changes. _Only_ those tests are run. So the first time you run this command, it will run every test. The second time (if you make no file changes), no tests will be run. If you change some files, just the tests which touch those files will be rerun. I use this command a lot!
- `run` -- Compiles (if necessary) and runs the `main` class in your _main_ source files. If there is no main class, you'll get an error. If there are multiple main classes, you'll be prompted to choose the one you want.
- `exit` -- Gets you out of Dodge (Ctrl-C also works!)
- `show <setting>` -- Prints out the value of the setting key specified by the `setting` parameter (without angle brackets). For example, `show name` should print out `funzles`.

Tab completion works, by the way! There are literally hundreds of available commands, of varying usefulness and specialization. Feel free to poke around. SBT doesn't have the ability to randomly break anything, and nothing you do in the SBT prompt (outside of the `save` command, which you should not use) will affect any of your source files. So it's safe to experiment!

### Creating, Compiling and Running

After installing your preferred Scala editor, create `Fun.scala` in the `src/main/scala/funzles/` directory containing:

```scala
package funzles

object Fun {
  def main(args: Array[String]): Unit = {
    println( "Hello, World!" )
  }
}
```

From within the SBT prompt, run the following command (you can also run `sbt run` from your shell prompt, but again, you will pay the startup penalty every time):

```scala
> run
```

SBT will compile and run your `Fun` class, printing _Hello, World!_ to standard output.

One very useful trick of SBT is the `~` prefix. You can prepend a `~` to _any_ SBT command. The result of this prefix is the command will run once, followed by SBT pausing and monitoring your project files. Whenever your source files change, it will immediately re-run the command, followed by another pause to monitor files for subsequent runs. This will continue until you break out of the `~` by using Enter. For example, run the following command:

```scala
> ~run
```

Now go back to your editor, and add the following line to the `main` method:

```scala
// ...
println( "...do you read me?" )
```

By the time you tab back to your terminal application, your class should have been recompiled and rerun, with the results waiting for you.

Once you get ramped up in Scala, you will find yourself using `~` almost all the time in SBT. It is particularly useful in conjunction with the `testQuick` command referenced earlier.

## Depending on Libraries

Before we close out, it's worth noting briefly how it is you can depend upon third-party libraries. The JVM has a very extensive ecosystem of libraries, built up over more than 20 years of community development. Scala can trivially access any Java library, and SBT is very much aware of the Java ecosystem.

Let's say that you want to do some low-level asynchronous network IO using the fantastic [Netty framework](https://netty.io/). You will want to look for the latest version at [search.maven.org](http://search.maven.org/). After a bit of digging, it looks like the latest Netty artifact (at the time of this writing) is [here](http://search.maven.org/#artifactdetails%7Cio.netty%7Cnetty-all%7C4.1.6.Final%7Cjar). Specifically, the full artifact descriptor is `io.netty:netty-all:4.1.6.Final`. We can instruct SBT to make this framework available to our project by modifying our build.sbt file:

```scala
name := "funzles"

scalaVersion := "2.11.8"

libraryDependencies += "io.netty" % "netty-all" % "4.1.6.Final"
```

If you still have the SBT prompt open (which you should!), run the `reload` command. This will cause SBT to pick up the latest changes to the build.sbt file. Now that you've done this, you can use Netty within your project sources as described in [their documentation](https://netty.io/wiki/user-guide-for-4.x.html).

Now, Netty is a Java framework. Scala frameworks can be added to our project using much the same mechanism, but the syntax and process is a little different. Let's imagine that we want to add a library for doing abstracted functional programming, similar to what we would get in Haskell out of the box. After a bit of googling, we might find that [Cats](https://github.com/typelevel/cats) is one such project. Visiting the project page on GitHub, we see early on in the README the following incantation (as of this writing):

```scala
libraryDependencies += "org.typelevel" %% "cats.core" % "2.0.0"
```

So all we need to do is add that to our build.sbt file, run another `reload` and we should be good to go!

```scala
name := "funzles"

scalaVersion := "2.11.8"

libraryDependencies += "io.netty" % "netty-all" % "4.1.6.Final"
libraryDependencies += "org.typelevel" %% "cats" % "0.7.2"
```

Most libraries written in Scala will have a line like this in their README, which makes it quite easy to add almost any Scala library to an SBT build without messing around at [search.maven.org](http://search.maven.org).

One thing you will notice is that this dependency uses _two_ percent signs (`%`) rather than just the one that we used with Netty. Without going into too much detail, this is part of SBT's workaround mechanism for the fact that binary compatibility in Scala can be a hard problem. As a general rule, if you're depending on a Java library, just use one `%`. If you're depending on a Scala library, use two.

At Rally, you won't see library dependencies declared in this way inside a `build.sbt` file because similar projects would duplicate the same settings. Instead we use a `project/Dependencies.scala` file by convention. These conventions are explained in the Quick Reference.

### Testing

Testing is incredibly important in all languages, which is why I'm not going to spend any time on it at all! :-) More seriously, Scala has two standard testing frameworks (with a third standard support framework used by both), and nearly code written in the language is tested using one of these two:

- [ScalaTest](http://www.scalatest.org/) -- A very broad test framework offering support for several testing paradigms (JUnit-style unit testing, RSpec-style BDD, and everything in between). ScalaTest supports a very large number of test definition styles, selected by the trait you inherit (e.g. `FunSpec`).
- [Specs2](http://etorreborre.github.io/specs2/) -- A slightly more narrowly-focused test framework which focuses on BDD. Specs2 supports two different test definition styles: mutable and immutable. I _highly_ recommend sticking with the mutable definition style, as it is more straightforward and far more commonly used in the Scala ecosystem.

Connect uses ScalaTest. Other teams' usage may vary. If in doubt, just ask someone else on your team.

## Next Steps

You're now up and running with a fully-functional, no-holds-barred Scala project environment. All the training wheels are off here: this is literally the same sort of environment (build setup, filesystem and editor) that nearly all Scala developers use on a daily basis! ![(smile)](https://wiki.audaxhealth.com/s/en_GB/7601/2b9cef12215b66d36aac7298ee6598161e5b8cbb/_/images/icons/emoticons/smile.png "(smile)")

Dual licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)/[CC BY 3.0](https://creativecommons.org/licenses/by/3.0/). Originally by Daniel Spiewak and updated/adapted for Rally use.
