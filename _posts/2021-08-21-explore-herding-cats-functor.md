---
title: "Explore \"herding cats\": Functor"
permalink: /cats/functor/
# header:
#   image: /assets/images/writerjpg.jpg
excerpt: "Explore Cats Functor. Understanding of \"contramap\" function"
categories:
  - cats
tags:
  - scala
classes: wide

---


Functor extends Invariant. 
```scala
trait Functor[F[_]] extends Invariant[F]
```

That's a little bit tricky once invariant functor is typically explained as mix of usual (covariant) and contravariant functors.
Those functors are something like:
```scala
trait CovariantFunctor[A]:
  def map[B](f: A => B): CovariantFunctor[B]


trait ContravariantFunctor[A]:
  def contramap[B](f: B => A): ContravariantFunctor[B]
```

With HKT precise definitions are going to be:
```scala
trait CovariantFunctor[F[_]]:
  def map[A, B](fa: F[A])(f: A => B): F[B]


trait ContravariantFunctor[F[_]]:
  def contramap[A, B](fa: F[A])(f: B => A): F[B]
```

Let's return back to the notion of Invariant.
[Cats Invariant doc](https://typelevel.org/cats/typeclasses/invariant.html) and [Softwaremill Invarian note](https://blog.softwaremill.com/scala-cats-invariant-functor-be57d2e2fa91) provide nice examples.
Nevertheless, I want to attempt to bring something extremely down-to-earth.

Let we have some thin wrapper:
```scala
trait EqWrapper[T]:
  def eqv(valueToCompare: T): Boolean
  def get: T
```

String implementation:
```scala
class StringEqWrapper(private val value: String) extends EqWrapper[String] {
  def eqv(valueToCompare: String): Boolean = valueToCompare == value
  def get: String = value
}
```

We can assume that in real world we define some non-trivial behaviour. It would be nice to have opportunity to derive new instances from old one using old ones as "back end".

Let's attempt to do it with usual `map`:
```scala
class StringEqWrapper(private val value: String) extends EqWrapper[String] { self =>
  def eqv(valueToCompare: String): Boolean = valueToCompare == value
  def get: String = value
  def map[T](f: String => T): EqWrapper[T] = new EqWrapper[T] {
    override def eqv(valueToCompare: T): Boolean = self.eqv("dummy") // I need T => String here to convert value to familiar strings
    override def get: T = f(self.get) //map of basic covariant functor works well
  }
}
```
At that point we see that `String => T` helped to implement `get`. We just apply function to underlying string value and return result.

But we can't compare `T` with `String`.

Contravariant approach leads to opposite result.
```scala
class StringEqWrapper2(private val value: String) extends EqWrapper[String] { self =>
  def eqv(valueToCompare: String): Boolean = valueToCompare == value
  def get: String = value
  def map[T](f: T => String): EqWrapper[T] = new EqWrapper[T] {
    override def eqv(valueToCompare: T): Boolean = self.eqv(f(valueToCompare)) // I know how to convert that T value to well-known String
    override def get: T = null.asInstanceOf[T] //I'm in trouble, I have String state but no idea how to return T value
  }
}
```
We can derive implement `eqv(valueToCompare: T)` to compare `T` with internal `String` state.

But `get` require something to convert internal `String` state to `T`.

```scala
class StringEqWrapper(private val value: String) extends EqWrapper[String] { self =>
  def eqv(valueToCompare: String): Boolean = valueToCompare == value
  def get: String = value
  def imap[T](f: String => T, g: T => String): EqWrapper[T] = new EqWrapper[T] {
    override def eqv(value: T): Boolean = self.eqv(g(value))
    override def get: T = f(self.get)
  }
}
```
Now we can derive new instance with `imap`:
```scala
val stringEqWrapper = new StringEqWrapper3("42")
val intEqWrapper: EqWrapper[Int] = stringEqWrapper.imap(_.toInt, _.toString)

intEqWrapper eqv 42 //true
```

[Typelevel Functor docs](https://typelevel.org/cats/api/cats/Functor.html) provides good description of API.

Explicitly I can denote that it is a right time to pay attention to [type lambdas](http://dotty.epfl.ch/docs/reference/new-types/type-lambdas-spec.html), there is also nice [rockthejvm post](https://blog.rockthejvm.com/scala-3-type-lambdas/) about it.
```scala
import cats.Functor
val listFuntor: Functor[List] = Functor[List]
listFuntor.as(List(1, 2, 3), "a") //List(a, a, a)

val listOfOptionFunctor: Functor[[α] =>> List[Option[α]]] = listFuntor.compose[Option] //Functor[λ[α => F[G[α]]]] in Scala2
listOfOptionFunctor.map(List(Some(1), None))("N" + _) //val res0: List[Option[String]] = List(Some(N1), None)
```
[Functor examples](https://github.com/antonkw/explore-herding-cats/blob/main/src/main/scala/io/github/antonkw/4_functor.worksheet.sc)
