---
layout: doc-page
title: "Type checking outline"
---

# Type checking outline

This outline intends to give an overview of how to type check the features in this proposal and the additional requirements to the compiler.

## Extension methods

Extension methods must not be placed in the top-level, instead they should only be placed in `object`s or type classes.
They will have slightly different semantics when encountered inside an `object` vs when encountered inside a type class.

## Type classes

Type classes are defined with the combination of the `type` and `class` keywords. They may have a number of type parameters and a number of type class requirements using those type parameters.
Their bodies may contain a number of abstract and non-abstract methods including extension methods.
Implementations of these methods are able to access all the methods defined by the type classes specified in the type class requirements.
Additionally, type classes can add default implementations for methods defined by required type classes using the `override` keyword.

Using extension methods inside type classes should generate a `trait` `Ops` with all of the extension methods the type class defines inside its companion object.


## Type class instances

Instances are introduced by creating implicit values of a type class.
These instances should be placed either in the companion object of the type class or the companion object of the data type for which the instance is created.
If this is not the case, the compiler should generate a warning.
Furthermore, if there's already an instance of the same type class for the same data type in scope, the compiler should generate an error and disallow it.

### Type class requirements

When defining an instance for a type class which defines type class requirements, the compiler should check if there are instances for all of the type class requirements in scope and otherwise generate an error.
Additionally if a type class has type class requirements and the type class instance itself has implicit instance dependencies (e.g. `extension eqForList[A: Order] = new Order[List[A]]`), then the compiler will need to check these dependencies if they fulfill the requirement for other instances that also specify implicit instance dependencies.


