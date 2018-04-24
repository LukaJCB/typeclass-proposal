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

This defines a `Semigroup` instance


```scala
implicit val intAdditionMonoid: Monoid[Int] = new Monoid[Int] {
  def empty: Int = 0
}
```

When defining an instance for `Monoid[Int]` above, an instance of `Semigroup[Int]` has to be in scope.
Which in turn means that we can use all the methods defined on `Semigroup` for creating our instance of `Monoid`.

We can also define type classes inductively, for example an `Option[A]` is a `Semigroup`, whenever `A` is also a `Semigroup`:


```scala
implicit val optionSemigroup[A: Semigroup]: Semigroup[Option[A]] = new Semigroup[Option[A]] {
  def combine(this x: Option[A])(y: Option[A]): Option[A] = (x, y) match {
    case (Some(a), Some(b)) => a.combine(b)
    case (_, _) => None
  }
}
```


## Type class coherence

With `opaque` types already implemented, coherence is a goal for this type class proposal.
In the context of type classes, coherence means that there should only be one instance of a type class for any given type.
Right now, failing to abide by type class coherence results in a compiler error, since the compiler isn't able to decide which instance you might want to use.

For instances to be coherent they have to be placed either in the companion object of the type class or the companion object of the data type the instance is defined for.
We should still allow local instances that can be brought in scope like they can today, but the compiler should at least emit a warning.

As an example, this is a valid instance decleration:

```scala
case class Foo[A](a: A, n: Int)

object Foo {
  implicit val fooFunctor: Functor[Foo] = new Functor[Foo] {
    def map[A, B](this fa: Foo[A])(f: A => B): Foo[B] =
      Foo(f(fa.a), fa.n)
  }
}
```

Whereas this one would emit a warning:

```scala
object Bar {
  implicit val fooFunctor: Functor[Foo] = new Functor[Foo] {
    def map[A, B](this fa: Foo[A])(f: A => B): Foo[B] =
      Foo(f(fa.a), fa.n)
  }
}
```


## Using default implementations

In the last section about defining type classes we spoke briefly about providing default implementations for methods on implied type classes.

```scala
implicit def readerApplicativeFunctor[R]: Applicative[[A] => Reader[R, A]] =
  new Applicative[[A] => Reader[R, A]] with Functor[A] => Reader[R, A]] {
    def pure[A](a: A): Reader[R, A] =
      Reader(_ => a)

    def ap[A, B](this ff: Reader[R, A => B])(fa: Reader[R, A]): Reader[R, B] =
      Reader(r => ff.run(r)(fa.run(r)))
  }
```

This will create both an instance of `Applicative` and `Functor` for `Reader`.
Note that now that a `Functor` instance for `Reader` is in scope, creating another one will result in an error.

Next we will look at how this scheme will work with types that have type class instances with requirements on the same type class instance.


// TODO this kinda breaks the whole expression thing, rethink this maybe?
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
This makes it clear where `Eq[Int]` comes from and leaves no ambiguities to the compiler.

## Multiple inductive instances

Sometimes we might be able to define different type class instances for a type constructor like `Option` whenever the type `A` we use within `Option[A]` has different instances.
For example we can define an instance of `Eq[Option[A]]` whenever `A` has an instance of `Eq`, and we can also define an instance of `Order[Option[A]]` whenever `A` has an instance of `Order`.
Using the inheritance based type class encoding we need to place these instances in different classes that extend from each other to avoid ambigiuous implicits.

With this proposal, that should no longer be an issue and we're able to define all of them in the same scope:

```scala
case class Foo[A](a: A)

object Foo {
  implicit def fooEq[A: Eq]: Eq[Foo[A]] = new Eq[Foo[A]] {
    def eqv(this x: Foo[A])(y: Foo[A]): Boolean =
      x.a.eqv(y.a)
  }

  implicit def fooOrder[A: Order]: Order[Foo[A]] = new Order[Foo[A]] {
    def compare(this x: Foo[A])(y: Foo[A]): Int =
      x.a.compare(y.a)
  }
}
```

Here we define a `Foo` case class and add an instance of `Eq` as well as an instance of `Order`.
The compiler can figure out that if `A` has an instance of `Order`, it must also have an instance of `Eq`, therefore it can tell that `Foo[A]` also has instance of `Eq` and therefore the definition of `Order[Foo[A]]` is fully legal.


## Coherence and multi-param type classes

//TODO