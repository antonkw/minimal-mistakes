---
title: "Explore \"herding cats\": writer"
permalink: /cats/writer/
header:
  image: /assets/images/writerjpg.jpg
excerpt: "Explore Cats Writer monad"
categories:
  - cats
tags:
  - scala
classes: wide

# toc: true
# toc_label: "My Table of Contents"
# toc_icon: "cog"
---

In that post we will check out the beginning of "herding cats" series by eed3si9n.


### Writer
[Scala with cats](https://books.underscore.io/scala-with-cats/scala-with-cats.html) says: `cats.data.Writer` is a monad that lets us carry a log along with a computation. We can use it to record messages, errors, or additional data about a computation, and extract the log alongside the final result.

#### Typing

Let's steal idea of `Logged` type from the book:
```scala
type Logged[A] = Writer[Vector[String], A]
```

On the left side we have our structure to keep logs.
On the right we have our `A` value. Hence, we can fix the type to persist log and sole generic will be our `A` type for values.

Under the hood we need `Monoid` for log lines. Reasoning for having full-featured `Monoid` is necessity to make logging optional and produce computation results without any logs at all. `Monoid[L].empty` helps with that demand. And we need `Applicative[F]` to pack values.
```scala
def liftF[F[_], L, V](
  fv: F[V]
)(
  implicit monoidL: Monoid[L], F: Applicative[F]
): WriterT[F, L, V] =
    WriterT(F.map(fv)(v => (monoidL.empty, v)))
```

#### Writer as WriterT parametrized with Id

Yo can notice that we started to deal with `WriterT[F, L, V]` instead of `Writer`.
`Writer` is an alias for `WriterT` with fixed [Id](https://typelevel.org/cats/datatypes/id.html) data type. 
```scala
type Writer[L, V] = WriterT[Id, L, V]
```

#### API

##### Run

We have `run` that triggers execution and "unpack" `WriterT`.
```scala
123.pure[Logged].run === (Vector(), 123)
```

##### Map

`map` allows tranformation of value.
```
Writer("example", 2).map(_ * 2).run === ("example", 4)
```
