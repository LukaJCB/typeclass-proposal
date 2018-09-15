---
layout: doc-page
title: "Instance Declarations"
---

# Instance Declarations

Declaring an instance for a type class is done by declaring an extension with a given name and then binding it to a new instance of the typeclass, not unlike how implicit-based type class instances are created today.
For example, 

```scala
extension intAdditionSemigroup : new Semigroup[Int] {
  def combine(this x: Int)(y: Int) = x + y
}
````

This defines a `Semigroup` instance.


```scala
extension intAdditionMonoid : new Monoid[Int] {
  def empty: Int = 0
}
```

When defining an instance for `Monoid[Int]` above, an instance of `Semigroup[Int]` has to be in scope.
Which in turn means that we can use all the methods defined on `Semigroup` for creating our instance of `Monoid`.

We can also define type classes inductively, for example an `Option[A]` is a `Semigroup`, whenever `A` is also a `Semigroup`:


```scala
extension optionSemigroup[A: Semigroup] : new Semigroup[Option[A]] {
  def combine(this x: Option[A])(y: Option[A]): Option[A] = (x, y) match {
    case (Some(a), Some(b)) => Some(a.combine(b))
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
  extension fooFunctor : new Functor[Foo] {
    def map[A, B](this fa: Foo[A])(f: A => B): Foo[B] =
      Foo(f(fa.a), fa.n)
  }
}
```

Whereas this one would emit a warning:

```scala
object Bar {
  extension fooFunctor : new Functor[Foo] {
    def map[A, B](this fa: Foo[A])(f: A => B): Foo[B] =
      Foo(f(fa.a), fa.n)
  }
}
```


## Using default implementations

In the last section about defining type classes we spoke briefly about providing default implementations for methods on implied type classes.

```scala
extension readerApplicativeFunctor[R] :
  new Applicative[[A] => Reader[R, A]] with Functor[A] => Reader[R, A]] {
    def pure[A](a: A): Reader[R, A] =
      Reader(_ => a)

    def ap[A, B](this fa: Reader[R, A])(ff: Reader[R, A => B]): Reader[R, B] =
      Reader(r => ff.run(r)(fa.run(r)))
  }
```

This will create both an instance of `Applicative` and `Functor` for `Reader`.
Note that now that a `Functor` instance for `Reader` is in scope, creating another one will result in an error.

Next we will look at how this scheme will work with types that have type class instances with requirements on the same type class instance.


```scala
extension intOrderEq : new Order[Int] with Eq[Int] {
  def compare(this x: Int)(y: Int): Int =
    if (x < y) -1 else if (x > y) 1 else 0
}

extension intHash : new Hash[Int] {
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
  extension fooEq[A: Eq] = new Eq[Foo[A]] {
    def eqv(this x: Foo[A])(y: Foo[A]): Boolean =
      x.a.eqv(y.a)
  }

  extension fooOrder[A: Order] = new Order[Foo[A]] {
    def compare(this x: Foo[A])(y: Foo[A]): Int =
      x.a.compare(y.a)
  }
}
```

Here we define a `Foo` case class and add an instance of `Eq` as well as an instance of `Order`.
The compiler can figure out that if `A` has an instance of `Order`, it must also have an instance of `Eq`, therefore it can tell that `Foo[A]` also has instance of `Eq` and therefore the definition of `Order[Foo[A]]` is fully legal.


## Coherence and multi-param type classes

The instances of Multi parameter type classes have a number of companion objects where their type shows up in the instance declaration.
This is problematic because it can mess with coherence.
For an example let's look at the `Collection` type class again:

```scala
type class Collection[C, E] {
  def insert(this collection: C)(elem: E): C
  def contains(this collection: C)(elem: E): Boolean
}
```

Now imagine we add an instance of `Collection[String, Char]` inside the `String` companion object.
Then someone else defines an instance of `Collection[CharList, Char]` in another package where `String` isn't in scope.
Then, when we try to use both packages together and ask for an instance of `Collection[_, Char]`, the compiler can't know which one we want, thereby forcing us to specify both types as to not break coherence.

Other languages overcome this problem with one of two techniques, functional dependencies or associated types.
Both of these solutions function by introducing a concept of determining a type by another.
For example, in the `Collection` type class above, the collection type always determines which type of element can be stored inside.
We say that `E` is uniquely determined by `C`.

In Rust only associated types are available and one can do something very similar in Scala with path dependent types. [Here](https://doc.rust-lang.org/book/first-edition/associated-types.html) is an article that describes the motivation and usage of associated types in Rust.

This [article by Miles Sabin](https://milessabin.com/blog/2011/07/16/fundeps-in-scala/) gives a good overview of how functional dependencies currently work in Scala and how they are currently used in the collections library. 


One compromise without bringing in full on support for functional dependencies could be to let the first parameter of the type class in question determine all other parameters.
In that case you would only be able to place instances in the companion object of the first parameter's type or the companion object of the type class itself.

Another approach could be to keep things as they are when using implicit based type classes, so that you can define instances for each set of types.
In which case it instances could go in either of type's companion objects or again the companion of the type class.

We should also mention bidirectional functional dependencies, though they are very rare and thus, do not need direct support in this proposal.

//TODO Decide on a scheme.
