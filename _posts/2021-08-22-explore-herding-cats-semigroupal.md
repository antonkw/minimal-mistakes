---
title: "Explore \"herding cats\": Semigroupal, Apply, Applicative"
permalink: /cats/apply/
# header:
#   image: /assets/images/writerjpg.jpg
excerpt: "Explore Cats Semigroupal, Apply, Applicative: product, ap, mapN and friends"
categories:
  - cats
tags:
  - scala
classes: wide

---

### Semigroupal

```scala
trait Semigroupal[F[_]] extends Serializable {
  def product[A, B](fa: F[A], fb: F[B]): F[(A, B)]
}
```
Here we're dealing with cartesian product.

```scala
Semigroupal[Option].product(1.some, 2.some) === (1,2).some
Semigroupal[Option].product(1.some, none[Int]) === none[(Int, Int)]
Semigroupal[List].product(List(1, 2, 3), List("foo", "bar")) === List((1, "foo"), (1, "bar"), (2, "foo"), (2, "bar"), (3, "foo"), (3, "bar"))
```

### Apply

Apply is quite tricky.

There are plenty of explanations that aren't bringing much sense to me.

Docs describe main function `def ap[A, B](ff: F[A => B])(fa: F[A]): F[B]` as "Given a value and a function in the Apply context, applies the function to the value."

That still far away from motivation since our usual friends like `List` or `Option` typically shouldn't handle function inside.

More or less good wording sounds like "apply lifts function `A => B` to container F[_]".

It's easy to construct such kind of composition where we build `Option[A => B]` and pass values there but that's not a tooling for everyday anyway.

[What is ap](https://typelevel.org/cats/typeclasses/applicative.html#what-is-ap) provides another one example.

```scala
val applyOption: Apply[Option] = Apply[Option]
val optionOfStringToUpperCase: Option[String] => Option[String] = applyOption.ap[String, String](((s: String) => s.toUpperCase).some)
val upper1 = optionOfStringToUpperCase("string".some)
upper1 === "STRING".some
optionOfStringToUpperCase(none[String]) === none

val toUpper: String => String = _.toUpperCase
val upper2 = toUpper.some <*> "string".some
upper2 === "STRING".some
```

`ap2` and `map2` are introduced here too.

```scala
Apply[Option].map2("hello ".some, "world".some)(_ + _) === "hello world".some
Apply[Option].map2(none[String], "world".some)(_ + _) === none[String]

val composeTwoOptions: (Option[String], Option[Int]) => Option[String] = Apply[Option].ap2(((s: String, i: Int) => s + i).some)
composeTwoOptions.apply("hi".some, 1.some) === "hi1".some
composeTwoOptions.apply("hi".some, none[Int]) === none[String]
```

Product left/right are important tools, and they're declared at Apply.

The allows to omit result of computations on the left/right side.
```scala
"hello".some *> "world".some === "world".some
"hello".some <* "world".some === "hello".some
none[String] *> "world".some === none[String]
none[String] <* "world".some === none[String]
```

#### mapN magic

`cats.ApplyArityFunctions` is responsible for bringing `map3`, `map4`, etc.

```scala
Apply[Option].map3(2.some, 2.some, 1.some)(_ + _ + _) === 5.some
```

`cats.syntax.TupleSemigroupalSyntax` brings some magic with `mapN`:
```scala
(1.some, 2.some).mapN { case (a, b) => a + b } === 3.some
```

Magic is multiplied by zero under the hood:
```scala
private[syntax] final class Tuple3SemigroupalOps[F[_], A0, A1, A2](private val t3: Tuple3[F[A0], F[A1], F[A2]]) {
  def mapN[Z](f: (A0, A1, A2) => Z)(implicit functor: Functor[F], semigroupal: Semigroupal[F]): F[Z] = Semigroupal.map3(t3._1, t3._2, t3._3)(f)
```
So, there are just dozens of implementations.

[Apply examples](https://github.com/antonkw/explore-herding-cats/blob/main/src/main/scala/io/github/antonkw/5_apply.worksheet.sc)

### Applicative

Typically `Applicative` is described as applicative functor where `map`, `ap`, and `pure` are equally important.

We already considered `Apply` and `Functor`, hence we're interested in `pure` method responsible for initialization of specified container: `def pure[A](a: A): F[A]`

For Either it is going to be `Right(a)`, Option has `Some(a)`, and so on.

Even while it seems extremely natural when we work with particular implementations it is vital to have abstraction to describe such a thing.

There are good definitions for `pure` and `product`: (Applicative Typeclass)[https://typelevel.org/cats/typeclasses/applicative.html#applicative]

```scala
Applicative[Option].pure(1) === 1.some
Applicative[Vector].pure(1) === Vector(1)
```

You can "replicate" values inside F:
```scala
Applicative[Option].replicateA(3, 1.some) === List(1, 1, 1).some
```


Applicative is composable:
```scala
Applicative[List].compose[Vector].compose[Option].pure(3) === List(Vector(3.some))
```


Some unit-functions that could be used when you need to preserve F-context but content doesn't matter or should be hidden:
```scala
Applicative[Option].unit === ().some

Applicative[List].whenA(true)(List(1, 2, 3)) === List((), (), ())
```