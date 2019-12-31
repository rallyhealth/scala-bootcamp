[/home/djo/code/backend-bootcamp/src/TODO.md]:
Invalid or missing YAML header/delimiters.  (Use 3 hyphens before and after.)

==============================================

# TODO

## New series'

### Topics on beginning to think in Scala

* Model data using traits and case classes
  * Traits allow us to apply a given protocol or API to diverse actual Class types.
    * An alternative to implicits for extending existing classes
      * Mix-in when an object is created (e.g.: `new java.io.File(".") with XFile`)
    * Describe abstract data shapes that can be implemented by many existing classes
      * ListLike, VectorLike, MapLike
* More on writing pure functions
  * The Obviously Pure
  * When implementation detail isn't pure
    * ...but it isn't visibly impure (If a tree falls in the forest and nobody sees it, it didn't happen)
    * `var` inside a function
    * output as an invisible side-effect
* Use the type system as executable program documentation
  * Option / Either / Try
* Use `for` comprehensions instead of `flatMap` / `map` when possible
  * On using the same data type throughout
  * Hints when one can't use the same data type throughout

