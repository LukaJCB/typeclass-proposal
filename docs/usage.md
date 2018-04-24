---
layout: doc-page
title: "Type class usage"
---

# Type class usage

We've already discussed one possible use case using the infix syntax described in the first section of this proposal.

```scala
type class Semigroup[A] {
  def combine(this x: A)(y: A): A

  def |+|(this x: A)(y: A): A = combine(x)(y)
}

object Semigroup {
  implicit val intSemigroup: Semigroup[Int] = new Semigroup[Int] {
    def combine(this x: Int)(y: Int): Int =
      x + y
  }
}



123 |+| 456
42.combine(-7)
```

This works well for functions that make use of extension methods, but some functions like `Monoid#empty` or `Applicative#pure` can't make use of this, as they don't operate on a value, but operate on the type instead.
To support syntax for these cases, we can call the instance directly when it's scope and access its methods from there:

```scala
val x: Int = Monoid[Int].empty

val y: List[Int] = Applicative[List].pure(x)
```



## Using type classes generically

Now we'll look at how we can use the type classes in a generic way.
If we want to write a function that requires a generic parameter `T` to have an instance of `Monoid`, we can define it to take an implicit instance of it, very similar to the way it is usually done with inheritance type classes today.

```scala
def sum[T](list: List[T])(implicit monoidT: Monoid[T]): T =
  list.foldLeft(Monoid[T].empty)(_ |+| _)
```


We could also define the same type class in terms of context bounds as it often done today as well:

```scala
def sum[T: Monoid](list: List[T]): T =
  list.foldLeft(Monoid[T].empty)(_ |+| _)
```