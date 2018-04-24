---
layout: doc-page
title: "Instance Declarations"
---

# Instance Declarations

Declaring an instance for a type class should work the same way as it does today for simple type classes.
That means creating an implicit value of the type class in question with the instance type.
For example, 

```scala
implicit val intAdditionSemigroup: Semigroup[Int] = new Semigroup[Int] {
  def combine(this x: Int)(y: Int) = x + y
}
````



```scala
implicit val intAdditionMonoid: Monoid[Int] = new Monoid[Int] {
  def empty: Int = 0
}
```

When defining an instance for `Monoid[Int]` above, an instance of `Semigroup[Int]` has to be in scope.
Which in turn means that we can use all the methods defined on `Semigroup` for creating our instance of `Monoid`.


## Type class coherence

With `opaque` types already implemented, coherence is a goal for this type class proposal.
In the context of type classes, coherence means that there should only be one instance of a type class for any given type.
Right now, failing to abide by type class coherence results in a compiler error, since the compiler isn't able to decide which instance you might want to use. 

//TODO Explain where instances have to be placed to be coherent


## Using default implementations

In the last section about defining type classes we spoke briefly about providing default implementations for methods on implied type classes.

```scala
implicit def readerApplicativeFunctor[R]: Applicative[[A] => Reader[R, A]] =
  new Applicative[[A] => Reader[R, A]] with Functor[A] => Reader[R, A]] {
    def pure[A](a: A): Reader[R, A] =
      Reader(_ => a)

    def ap[A, B](ff: Reader[R, A => B])(fa: Reader[R, A]): Reader[R, B] =
      Reader(r => ff.run(r)(fa.run(r)))
  }
```

This will create both an instance of `Applicative` and `Functor` for `Reader`.
Note that now that a `Functor` instance for `Reader` is in scope, creating another one will result in an error.

Next we will look at how this scheme will work with types that have type class instances with requirements on the same type class instance.

```scala
implicit val intOrderEq: Order[Int] = new Order[Int] with Eq[Int] {
  def compare(this x: Int)(y: Int): Int =
    if (x < y) -1 else if (x > y) 1 else 0
}

implicit val intHash: Hash[Int] = new Hash[Int] {
  def hash(this x: Int): Int = x.hashCode()
}
```

We define `Int` instances for `Order` as well as for `Hash` and `Eq`, where the `Eq` is implemented in terms of `Order`.

## Coherence and multi-param type classes

//TODO