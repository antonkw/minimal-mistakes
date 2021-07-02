---
title: "Explore \"herding cats\": Eq, Order, and why greatest element of set is not equal to maximal"
permalink: /cats/eq-order/
excerpt: "Explore Cats Eq, Partial Order, Order typeclasses"
categories:
  - cats
tags:
  - scala
# classes: wide

toc: true
toc_label: "My Table of Contents"
toc_icon: "cog"
header:
  teaser: https://content.onliner.by/widget/news/1x1/57763665a6cfb6c8df49d12994824458.jpeg
---

In that post we will check out the beginning of "herding cats" series by eed3si9n.

### Eq
[eed3si9n.com/herding-cats/Eq](https://eed3si9n.com/herding-cats/Eq.html)

Nothing special here, just can notice that anonymous `given` works well. With Scala 3 we can finally omit names like:
```scala
given Eq[IdCard] with {
  def eqv(a: IdCard, b: IdCard): Boolean = ???
}
```
No need to write `given idCardEq: Eq[IdCard]` with clear understanding that `idCardEq` won't be used directly.

Finally, it's always quite tempting to put *Eq* into companion object:
```scala
case class IdCard(firstName: String, secondName: String)

object IdCard:
  given Eq[IdCard] = Eq.fromUniversalEquals
```
[Eq example](https://github.com/antonkw/explore-herding-cats/blob/main/src/main/scala/io/github/antonkw/1_eq.worksheet.sc)

### PartialOrder
```scala
trait PartialOrder[A] extends Eq[A]
```

Eugene didn't pay much attention to that type class on the page [PartialOrder.html](https://eed3si9n.com/herding-cats/PartialOrder.html)

[Pos.html](https://eed3si9n.com/herding-cats/Pos.html) page about partially ordered sets doesn't bring much information too.

I recommend to take a look at least at wiki page: [wiki/Partially_ordered_set](https://en.wikipedia.org/wiki/Partially_ordered_set)

PartialOrder is all about notion of partial ordered sets.

* **Greatest** element and **least** element: An element g in P is a greatest element if for every element a in P, a ≤ g. An element m in P is a least element if for every element a in P, a ≥ m. A poset can only have one greatest or least element.
* **Maximal** elements and **minimal** elements: An element g in P is a maximal element if there is no element a in P such that a > g. Similarly, an element m in P is a minimal element if there is no element a in P such that a < m. If a poset has a greatest element, it must be the unique maximal element, but otherwise there can be more than one maximal element, and similarly for least elements and minimal elements.

That's quite interesting how different ideas of greatest and maximal elements.

Naive implementations are:
```scala
def greatest[A](
  set: Set[A]
)(
  using partialOrdering: PartialOrder[A]
): Option[A] = 
  set.find(
    elem => set.forall(
      partialOrdering
        .tryCompare(_, elem)
        .exists(_ <= 0)
    )
  )

def maximals[A](
  set: Set[A]
)(
  using partialOrdering: PartialOrder[A]
): Set[A] =
  set.filter(
    elem => set.forall(
      partialOrdering
      .tryCompare(_, elem)
      .map(_ <= 0)
      .getOrElse(true)
    )
  )

def least[A](
  set: Set[A]
)(
  using partialOrdering: PartialOrder[A]
): Option[A] =
  set.find(
    elem => set.forall(
      partialOrdering
        .tryCompare(_, elem)
        .exists(_ >= 0)
    )
  )

def minimals[A](set: Set[A])(using partialOrdering: PartialOrder[A]): Set[A] =
  set.filter(
    elem => set.forall(
      partialOrdering
        .tryCompare(_, elem)
        .map(_ <= 0)
        .getOrElse(true)
    )
  )
```

I also combined values to case class. So we'll have all properties at one place.
```scala
case class PosetDescription[A](
  greatest: Option[A], 
  least: Option[A], 
  minimals: Set[A], 
  maximals: Set[A]
)
```

[PartialOrder example](https://github.com/antonkw/explore-herding-cats/blob/main/src/main/scala/io/github/antonkw/2_partial.worksheet.sc)

It seemed that it is easy to abstract out sets itself to write something like:
```scala
def make[F[_]: Foldable, A](
  poset: F[A]
)(
  using partialOrdering: PartialOrder[A]
) = PosetDescription(
  greatest = poset.find(el => poset.forall(_.tryCompare(el).exists(_ <= 0)))
...
```
In fact, we're interested in the properties of set itself instead of *just* provided API.

There is also interesting idea of bounds.
For a subset *A* of *P*, an element *x* in *P* is an upper bound of *A* if *a* ≤ *x*, for each element *a* in *A*. In particular, ***x* need not be in *A*** to be an upper bound of *A*. Similarly, an element *x* in *P* is a lower bound of *A* if *a* ≥ *x*, for each element *a* in *A*. A greatest element of *P* is an upper bound of *P* itself, and *a* least element is a lower bound of *P*.
Nevertheless, writing code to decouple strongly connected components is out of scope. Let's move forward.

### Order

```scala
trait Order[A] extends PartialOrder[A]
```

Unlike `PartialOrder` we can meet `Order[A]` everywhere.
We can't write just:
```scala
val persons = NonEmptySet.of(john, anton)
```

```scala
no implicit argument of type cats.kernel.Order[io.github.antonkw.IdCard] was found for parameter A of method of in object NonEmptySetImpl
```

Signature is following: `def of[A](a: A, as: A*)(implicit A: Order[A]): NonEmptySet[A]`

Hence, we need to implement order:
```scala
given Order[IdCard] = Order.from((id1, id2) => {
  val initial = id1.firstName compare id2.firstName
  if (initial == 0) id1.secondName compare id2.secondName
  else initial
})
```

[Order example](https://github.com/antonkw/explore-herding-cats/blob/main/src/main/scala/io/github/antonkw/3_order.worksheet.sc)
