---
title: "Iron updates: turning opaque types into value objects"
permalink: /scala/iron-updates/
excerpt: "Next Iron release brings very important updates. There will be a tooling to turn opaque types into value objects (like newtype) with optional validation."
categories:
  - scala
tags:
  - scala
  - validation
  - newtype
  - iron
  - functional programming
classes: wide
---

<img alt="iron" src="/assets/images/logo_iron.png" width="300px">
Disclaimer.

It is not a full-featured introduction to Iron library.

The note highlights a couple of updates that seem very important to me.
The updates are not released yet, and even one unassigned "first good issue" ticket could be a good thing to tackle for somebody.


# Status quo
I need to write down some words for those of us who, like me, occasionally write in Scala 3 and look for the upgrade with curiosity.

The context for the note is co-allocations of the facts:
- Scala 3 doesn't support Scala 2 macro, macros-based features of [scala-newtype](https://github.com/estatico/scala-newtype) and [refined](https://github.com/fthomas/refined) libraries are not working. It definitely hurt people who used to have everything neat with definitions like this:
  ```scala
  @newtype case class FirstName(value: NonEmptyString)
  ```
- `opaque` types were introduced. Initially, they were perceived (by some people, including me) as built-in tooling to have a native mechanism for Value Objects (in DDD terms). In practice, additional manipulations are required; opaque types are not equivalent to newtypes. Specifically, extra tricks are needed to enable standard `apply` and `value` functions. [trading/Newtype.scala](https://github.com/gvolpe/trading/blob/main/modules/domain/shared/src/main/scala/trading/Newtype.scala) is an example of how to turn on value objects back.

In case you aspire to acquire a richer understanding of the prior status quo and problems, I recommend reading the discussion: [Improve opaque types (scala-lang.org)](https://contributors.scala-lang.org/t/improve-opaque-types/4786/43)
[Gabriel Volpe](https://twitter.com/volpegabriel87) triggered `newtype`-specific discussion.
And more detailed motivation can be found in the original [SIP-35 - OPAQUE TYPES](https://docs.scala-lang.org/sips/opaque-types.html).

# Iron

## Brief intro
I recommend quickly going through the README/docs of [Iltotore/iron](https://github.com/Iltotore/iron) library.

Gabriel also has a short demo under the Trading project: [IronDemo.scala](https://github.com/gvolpe/trading/blob/main/modules/x-demo/src/main/scala/demo/IronDemo.scala):

```scala
type AgeR = DescribedAs[
  Greater[0] & Less[151],
  "Alien's age must be an integer between 1 and 150"
]
```

Long story short, you can define limitations as predicates and turn them into types. And convenient tooling help to perform validation against those types.

## Updates

### RefinedTypesOps

Let us have the definition:
```scala
opaque type Temperature = Double :| Positive
object Temperature extends RefinedTypeOps[Temperature]
```

First of all, now we have `apply` for cases when validation could be performed at compile time.
```scala
val temperature = Temperature(100)
```

Most likely, real usage will be the following:
```scala
val temperature = Temperature.either(runtimeVariable)
```
The point here is that a convenient companion object will pack the value into `opaque` type.

The neat detail I want to mention is the implication of constraints.
```scala
val x: Double :| Positive = 5.0
val y: Double :| Greater[10] = 15.0
val t1 = Temperature.fromIronType(x)
val t2 = Temperature.fromIronType(y)
```
`Double :| Greater[10]` is not of the same type but it is positive and could be used to create a `Temperature` without explicit validation.

### implementation insights
[Raphaël Fromentin](https://github.com/Iltotore) nudged me to use [Match Types](https://dotty.epfl.ch/docs/reference/new-types/match-types.html).

So, `RefinedTypeOps[T]` has a single parameter and looks for existing `IronType` without explicitly mentioning parameters for `IronType` itself! Very neat, and no macros are required.
```scala
type RefinedTypeOps[T] = T match
  case IronType[a, c] => RefinedTypeOpsImpl[a, c, T]
```

And `RefinedTypeOpsImpl` handles some functions:
```scala
class RefinedTypeOpsImpl[A, C, T]:
  inline def apply(value: A)(using Constraint[A, C]): T =
    autoRefine[A, C](value).asInstanceOf[T]
...
```

### `value`!
Reminder, `Temperature` is opaque type. So, by default, you can't access the underlying value. So, you know that there is a `Double` inside, but you have no access.
```scala
val t1 = Temperature(100)
val t2 = Temperature(100)

val result = t1.value + t2.value
```
`value` is enabled by extension for iron types with companion objects.
The cool thing here is that `value` returns the underlying `IronType`, and that type is the subtype of the underlying primitive.
- `t1.value + t2.value` is a legitimate operation considered the sum of two doubles. No `.value.value` calls to reach primitives!
- `t1.value` itself is `Double :| Positive` and can be used appropriately. E.x. use `fromIronType` with inherent implication to re-pack into different iron types.

### `NewType`?

With all those features enabled, we can use iron as convenient tooling to build lightweight value objects on top of the opaque types.
The library focuses on precise validation, so better language should be derived.
`True` predicate (which is always valid) is in place already.

```scala
opaque type FirstName = String :| True
object FirstName extends RefinedTypeOps[FirstName]
```
And `apply` works for any value in runtime!
```scala
val firstName = FirstName(anyRuntimeString)
```

The next potential improvements are:
- Provide an alias for `True` to depict that it is not a constraint but a tool to build value objects without validation: `opaque type FirstName = String :| Pure`
- Make an alias for `A :| Pure`. E.x. `opaque type FirstName = NewType[String]`.

The first option is convenient for initial domain modeling with the connotation of a placeholder that will be replaced later.
And `NewType` simply explicitly says "no validation here, just value object"

I suggest jumping into [iron/Issue#125: Add alias for True constraint and IronType[A, True]](https://github.com/Iltotore/iron/issues/125) to share thoughts or implement it!

Thank you for reading!Thank you for reading!