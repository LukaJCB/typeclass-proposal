---
layout: doc-page
title: "Type class definitions"
---


# Type class definitions

In this section we are going to into detail into the specific feature set type classes in Dotty should support. Type classes are declared by combining the already existing keywords `type` and `class`.

```scala
type class Semigroup[A] {
  def combine(x: A, y: A): A
}
```

Semantically their definition is almost identical to `trait`s, they can introduce new type parameters and they can also have a number of methods defined on them, which don't necessarily need an implementation.


## Combination with extension methods

To be able to utilize convenient syntax with our type classes, we could use the extension methods defined in the last section:

```scala
type class Semigroup[A] {
  def combine(x: A, y: A): A
}

object Semigroup {
  extension def combine[A: Semigroup](this x: A)(y: A): A
}
```

However that could still be considered a bit much boilerplate. 
We should be able to define the syntax right there in the type class declaration:

```scala
type class Semigroup[A] {
  extension def combine(this x: A)(y: A): A
}
```


This should also work for the `Traverse` type class we talked about in the prior section:

```scala
type class Traverse[F[_]] {
  extension def traverse[G[_]: Applicative, A, B](this fa: F[A])(f: A => G[B]): G[F[B]]

  extension def sequence[G[_]: Applicative, A](this fga: F[G[A]]): G[F[A]]

  extension def flatSequence[G[_], A](this fgfa: F[G[F[A]]])(implicit G: Applicative[G], F: FlatMap[F]): G[F[A]]

  //...
}
```

## Type class requirements (inheritance)

Unlike `trait`s we don't want type classes to be able to extends from another in the classical subtyping way.
Type classes can define requirements, that instances of a specific type class should also be instances of other type classes.
For example here is the definiton of `Monoid` and `Semigroup` to show this relationship. 

```scala
type class Semigroup[A] {
  extension def combine(this x: A)(y: A): A
}

type class Monoid[A] : Semigroup[A] {
  def empty: A
}
```

It should not be a direct inheritance relationship, instead it should be understood as `Monoid[A]` implies a `Semigroup[A]`.
This relationship doesn't express that every `Monoid[A]` is also a `Semigroup[A]`, but that whenever you want to create an instance of `Monoid[A]`, there must also be a `Semigroup[A]`.

This would allows us to have 

```scala
type class Eq[A] {
  extension def ===(this x: A)(y: A): Boolean
}

type class Order[A] : Eq[A] {
  extension def compare(this x: A)(y: A): Int
}

type class Hash[A] : Eq[A] {
  extension def hash(this x: A): Int
}
```

Adelbert Chang wrote a paper describing in great detail why subtyping for type classes is a suboptimal approach.
You can find this paper [here](https://adelbertc.github.io/publications/typeclasses-scala17.pdf).

Of course it should also be possible to require multiple other type classes:

```scala
type class CommutativeSemigroup[A] : Semigroup[A] {
  //...
}

type class CommutativeMonoid[A] : CommutativeSemigroup[A] : Monoid[A] {
  // ...
}
```



### Default implementations


One of the benefits of the current inheritance-based type class encoding in Scala is that it allows us to define default implementations for type classes higher up in the hierarchy.
The standard example for this would be the `Functor`-`Applicative`-`Monad` hierarchy.

```scala
trait Functor[F[_]] {
  extension def map[A, B](this fa: F[A])(f: A => B): F[B]
}

trait Applicative[F[_]] extends Functor[F] {
  extension def ap[A, B](this fa: F[A])(ff: F[A => B]): F[B]
  def pure[A](a: A): F[A]

  override extension def map[A, B](fa: F[A])(f: A => B): F[B] =
    ap(fa)(pure(f))
}

trait Monad[F[_]] extends Applicative[F] {
  extension def flatMap[A, B](fa: F[A])(f: A => F[B]): F[B]

  override extension def map[A, B](fa: F[A])(f: A => B): F[B] =
    flatMap(fa)(a => pure(f(a)))

  override extension def ap[A, B](fa: F[A])(ff: F[A => B]): F[B] =
    flatMap(ff)(f => map(fa)(f))
}
```

With these default implementations through overriding it's possible to implement only `flatMap` and `pure` and getting a valid `Monad` instance with `ap` and `map` derived.

Our proposal should be able to have the same ability.
In the same way as the above snippet, you can add default instances by using the existing `override` keyword.
This will have some consequences for defining instances which we will see later.
The following is how default implementations would look like in this proposal:

```scala
type class Functor[F[_]] {
  extension def map[A, B](this fa: F[A])(f: A => B): F[B]
}

type class Applicative[F[_]] : Functor[F] {
  extension def ap[A, B](this fa: F[A])(ff: F[A => B]): F[B]
  def pure[A](a: A): F[A]

  override extension def map[A, B](this fa: F[A])(f: A => B): F[B] =
    ap(fa)(pure(f))
}

type class Monad[F[_]] : Applicative[F] {
  extension def flatMap[A, B](this fa: F[A])(f: A => F[B]): F[B]

  override extension  def map[A, B](this fa: F[A])(f: A => B): F[B] =
    flatMap(fa)(a => pure(f(a)))

  override extension  def ap[A, B](this fa: F[A])(ff: F[A => B]): F[B] =
    flatMap(ff)(f => map(fa)(f))
}
```



## Multi-parameter type classes

Multi-parameter type classes are an important part of existing type class hierarchies, but also bring more complexities with them.
We will look into those in detail in a later section, for now let's look at an example of a simple multiparameter type class:

```scala
type class Collection[C, E] {
  extension def insert(this collection: C)(elem: E): C
  extension def contains(this collection: C)(elem: E): Boolean
}
```

In this example we have a type class that has a type parameter for the collection type as well as the element type.
Possible instances for this type class could be `Collection[List[A], A]` or `Collection[String, Char]`.

It's also possible to require multiple other type class instances.
For example we might want to require our `C` type to form a `Monoid` and our `E` type to be comparable using `Eq`.

```scala
type class Collection[C, E] : Monoid[C] : Eq[E] {
  extension def insert(this collection: C)(elem: E): C
  extension def contains(this collection: C)(elem: E): Boolean
}
```
