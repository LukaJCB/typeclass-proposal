# typeclass-proposal

This proposal is based on the [dotty proposal](https://github.com/lampepfl/dotty/pull/4153) by Martin Odersky and tries to address some of the perceived drawbacks from the functional programming Scala community. Therefore the rationale for this proposal should remain the same as the original one.

### Rationale
There are two dominant styles of structuring Scala programs: Standard (object-oriented) class hierarchies or typeclass hierarchies encoded with implicit parameters. Standard class hierarchies lead to simpler code and allow to dispatch on the runtime type, which enables some optimizations that are hard to emulate with implicits. Typeclass hierarchies are more flexible: instances can be given independently of the implementing types and the implemented interfaces, and instances can be made conditional on other typeclass instances.

Unfortunately, typeclass-oriented programming has a high upfront cost. It starts with the definitions of "typeclasses" themselves. Say you want to implement a typeclass for a map method (often called a Functor). The usage you want to support is:
```scala
xs.map(f)
```
However, you can't define map as a unary method like this, not if it is part of a typeclass. Instead you need to define a binary version of map, along the following lines:

```scala
trait Functor[F[_]] {
  def map[A, B](f: A => B, xs: F[A]): F[B] = ...
}
```
You need another implicit class, to get back map as an infix operator. Something like this:

```scala
implicit class InfixMap[F[_]: Functor, A](private val x: F[A]) extends AnyVal {
  def map(f: A => B): F[B] = implicitly[F].map(f, x)
}
```

It does the job, but at a cost of lots of boilerplate! The required boilerplate is very technical, advanced, and, I believe, frightening for a newcomer. The complexity of typeclasses does not end with their definition, either. It continues to the use sites, which typically need one extra type parameter per typeclass argument. Scala is a language set out to eliminate boilerplate and promote the simplest possible style of expression. But it seems in this area it has utterly failed to do so.

https://github.com/mpilquist/simulacrum is macro library that removes a lot of the boilerplate. But it relies on "macro paradise" which has never been officially supported and has lost its maintainer https://contributors.scala-lang.org/t/stepping-down-as-the-maintainer-of-scalamacros-paradise/1703. The kind of annotation macros required by simulacrum will almost certainly only be supported in Dotty as code generation tools, requiring a separate build step.

The aim of this proposal is to

- make typeclasses easy and pleasant to use.
- support use cases currently utilized by functional programming communities in Scala.
- in conjunction with opaque types, replace the lower-level constructs of value classes and implicit classes.

### Details
The proposal is written up as a list of reference pages:

1. [Extension Methods](docs/extensions.md)
2. [Type class definitions](docs/typeclasses.md)
3. [Instance Declarations](docs/instances.md)
4. [Type class usage](docs/usage.md)
5. [Translation Scheme](docs/translations.md)
