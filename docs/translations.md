---
layout: doc-page
title: "Translation scheme"
---


# Translation scheme

Ideally extension methods and type class instances along with opaque types should be full replacements for value classes and implicit classes.
Nevertheless it might be useful to look at translation schemes that provide translations from the constructs in this proposal to constructs that exist today.

## Extension methods

Extension methods could be translated to existing `implicit classes`:

Assuming an extension method of the form:

```
def <id> <type-params> (this <this-param> <this-type>)<params> <implicit-params>: <return-type> = <impl>
```

it can be translated to the following implicit class definition:


```
implicit class <id>Ops <type-params> (private val <this-param>: <this-type>) extends AnyVal {
  def <id> <params> <implicit-params>: <return-type> = <impl>
}
```

For example, this:

```scala
def concat[T](this x: List[T])(y: List[T]): List[T] = x ++ y
```

Would be translated to this:

```scala
implicit class concatOps[T](private val x: List[T]) extends AnyVal {
  def concat(y: List[T]): List[T] = x ++ y
}
```


## Type class translations

For a simple type class consider this translation scheme:

Assuming a type class defintion of the form:

```
type class <id> <type-params> {
  <defs>
}
```

Where `<defs>` is zero or more of either a normal method or an extension method with or without an implementation.
We can translate it into the following `trait`:


```
trait <id> <type-params> {
  <defs>
}
```

Where the extension methods inside `<defs>` are replicated into the companion object of the type class with the translation scheme for extension methods applied.
For example, this `Eq` type class:



```scala
type class Eq[T] {
  def eqv(x: T, y: T): Boolean
  def ===(this x: T)(y: T): Boolean = eqv(x, y)
}
```

Would be translated into this:

```scala
trait Eq[T] {
  def eqv(x: T, y: T): Boolean
  def ===(x: T)(y: T): Boolean = eqv(x, y)
}

object Eq {
  implicit class Eq===Ops[T](private val x: T) extends AnyVal {
    def ===(y: T)(implicit ev: Eq[T]): Boolean = ev.eqv(x, y)
  }
}
```

//TODO this scheme doesn't work if `Eq._` isn't in scope