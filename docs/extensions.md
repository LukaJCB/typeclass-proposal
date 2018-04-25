---
layout: doc-page
title: "Extension Methods"
---


# Extension Methods


Extension methods are a great way to add functionality to a type after it's been defined.
Here is a simple example:


```scala
object Ops
  def **(this i: Int)(j: Int): Int = Math.pow(i, j)
}

//whenever `Ops` is in scope
2 ** 3 == 8
```

This adds a simple exponent operator to integers. They should be able to be invoked exactly like calling a method on `Int`.
We introduce such an extension method by prefixing one of the parameters of the method with `this`.

## Advantages over implicit classes

When defining `Ops` for type classes that involve higher kinded types `F[_]`, we usually have operations that don't work on any type `F[A]`, but have specific requirements. 
For an example let's look at the `Traverse` type class:

```scala
trait Traverse[F[_]] {
  def traverse[G[_]: Applicative, A, B](fa: F[A])(f: A => G[B]): G[F[B]]

  def sequence[G[_]: Applicative, A](fga: F[G[A]]): G[F[A]]

  def flatSequence[G[_], A](fgfa: F[G[F[A]]])(implicit G: Applicative[G], F: FlatMap[F]): G[F[A]]

  //...
}
```

Ideally we'd want to call these 3 functions with infix syntax, i.e. `fga.sequence`.
With implicit classes we now need to define three different classes for these functions.
The extension methods proposal would make this definition a lot simpler:

```scala
object Traverse {
  def traverse[F[_]: Traverse, G[_]: Applicative, A, B](this fa: F[A])(f: A => G[B]): G[F[B]]

  def sequence[F[_]: Traverse, G[_]: Applicative, A](this fga: F[G[A]]): G[F[A]]

  def flatSequence[F[_]: Traverse: FlatMap, G[_], A](this fgfa: F[G[F[A]]]): G[F[A]]
}
```


