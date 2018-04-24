---
layout: doc-page
title: "Type class definitions"
---


# Type class definitions

In this section we are going to into detail into the specific feature set type classes in Dotty should support.




## Combination with extension methods

To be able to utilize convenient syntax with our type classes, we could use the extension methods defined in the last section:

```scala
type class trait Semigroup[A] {
  def combine(x: A, y: A): A
}

object Semigroup {
  implicit object Ops {
    def combine[A: Semigroup](this x: A)(y: A): A
  }
}
```

However that could still be considered a bit much boilerplate.


This should also work for the `Traverse` type class we talked about in the prior section:

```scala
type class trait Traverse[F[_]] {
  def traverse[G[_]: Applicative, A, B](this fa: F[A])(f: A => G[B]): G[F[B]]

  def sequence[G[_]: Applicative, A](this fga: F[G[A]]): G[F[A]]

  def flatSequence[G[_], A](this fgfa: F[G[F[A]]])(implicit G: Applicative[G], F: FlatMap[F]): G[F[A]]

  //...
}
```

## Type class requirements

```scala
type class trait Semigroup[A] {
  def combine(this x: A)(y: A): A
}

type class trait Monoid[A] : Semigroup[A] {
  def empty: A
}
```

It should not be a direct inheritance relationship, instead it should be understood as `Monoid[A]` implies a `Semigroup[A]`.


```scala
type class trait Eq[A] {
  def ===(this x: A)(y: A): Boolean
}

type class trait Order[A] : Eq[A] {
  def compare(this x: A)(y: A): Int
}

type class trait Hash[A] : Eq[A] {
  def hash(this x: A): Int
}
```

Adelbert Chang wrote a paper describing in great detail why subtyping for type classes is a suboptimal approach.


### Default implementations







## Multi-parameter type classes