---
title: "Reader is not just a function, isn't it?"
permalink: /cats/reader/
# header:
#   image: /assets/images/writerjpg.jpg
excerpt: "Is Reader is just a synonym to function?"
categories:
  - cats
tags:
  - scala
  - scala cats
  - cats reader
  - scala reader
  - reader monad
  - functional programming
classes: wide

---

That post will be devoted to the notion of Reader monad and its simplest application.
My original goal was just a careful exploration of dependency management in ZIO-way. `ZIO[R,E,O]` has an explicit Reader-channel (`R` stands for **R**eader) to declare necessary dependencies for any computation. That idea encourages me a lot. I love the inclination to explain all possible prerequisites and limitations for any piece of instructions.
The first thing I need to confide in is that I underestimated Reader monad as an abstraction. Reader monad for me has always sounded like a "clever name for simple function with additional chaining options" and nothing more.
Whilst I was going to jump into the comparison of Reader and class constructors and proceed with ZIO, I decided to re-iterate Reader motivation from scratch and move to real-like usage in the upcoming note.
My journey started from [Reader datatype (by eed3si9n)](https://eed3si9n.com/herding-cats/Reader.html) which particlarly mentions the talk 
[Rúnar Óli Bjarnason: Dead-Simple Dependency Injection](http://functionaltalks.org/2013/06/17/runar-oli-bjarnason-dead-simple-dependency-injection/).


It simply brings the notion of passing dependencies with Reader monad and exposing Reader as API for application services. Eventually, that note became the text version of the talk. I hope my explicit comments could help to ascertain some non-obvious aspects.

#### table
Let we have table with user-password combinations.
```sql
CREATE TABLE public.users (
	name text NOT NULL,
	secret_hash text NOT NULL,
	CONSTRAINT users_un UNIQUE (name)
);
```

#### initial function
At the very first step we have function that initiates connection, updates password, and finally closes connection.
```scala
def setPassword(user: String, hash: String) = {
  Class.forName("org.postgresql.Driver")
  val connection = DriverManager.getConnection("jdbc:postgresql://localhost:5432/antonkw", properties)val statement = connection.prepareStatement("update users set secret_hash = ? where name = ?")statement.setString(1, hash)
  statement.setString(2, user)
  statement.executeUpdate()
  connection.close()
}
```

#### factory for connections
It sounds sensible to not mix logic and management of connections.
```scala
def setPassword(user: String, hash: String) = {
  val connection = ConnectionFactory.getConnection
  val statement = connection.prepareStatement("update users set secret_hash = ? where name = ?")statement.setString(1, hash)
  statement.setString(2, user)
  statement.executeUpdate()
}
```
Key negative points:
- connection's lifycycle is out of control (who ought to close the connection?)
- dependency that not explicitly stated
- ConnectionFactory should be initialized previously and reasoning starts to spread out of the piece we're working on

#### inversion of control 
We can finally get rid of initialization inside our tiny function.
```scala
def setPassword(user: String, hash: String, connection: Connection) = {
  val statement = connection.prepareStatement("update users set secret_hash = ? where name = ?")statement.setString(1, hash)
  statement.setString(2, user)
  statement.executeUpdate()
}
```
We know that connection will be managed at the higher-level. Therefore, we allow ourselves to care only about ongoing task.
"Higher level" could be as simple as flat function that manages connection by itself:
```scala
def connectAndSetPassword(user: String, hash: String) = {
  Class.forName("org.postgresql.Driver")
  val connection = DriverManager.getConnection("jdbc:postgresql://localhost:5432/antonkw", properties)setPassword(user, hash)
  connection.close()
}
```

In fact, even our function could become a "higher level" function that should "feed" connection.
```scala
def setPassword(user: String, hash: String, connection: Connection) = {
  val statement = connection.prepareStatement("update users set secret_hash = ? where name = ?")statement.setString(1, hash)
  statement.setString(2, user)
  statement.executeUpdate()
  savePasswordChangeReport(user, connection)
}
```
`setPassword` should be also called after some checks that require connection.
What does it mean?
While it is possible to fix convention that lifecycle is managed by some factory it still should be passed back-and-forth across the layers of logic.

#### currying
We can get rid of connection as an argument. Currying allows us to define a builder for function that consumes connection.
```scala
def setPassword(user: String, hash: String): Connection => Unit =
  connection => {
    val statement = connection.prepareStatement("update users set secret_hash = ? where name = ?")
    statement.setString(1, hash)
    statement.setString(2, user)
    statement.executeUpdate()
  }
```
The strength here is delaying of passing the actual connection.
It sounds like our goal is possibility to compose more functions and finally pass connection once.
Notwithstanding, at that step we have no native tooling for that composition. 
Let we have more functions written with currying:
```scala
def savePasswordChangeReport(user: String, hash: String): Connection => Unit
``` 
We can't leverage "postponed" connection transmission:
```scala
def setPassword(user: String, hash: String): Connection => Unit =
  connection => {
    val statement = connection.prepareStatement("update users set secret_hash = ? where name = ?")
    statement.setString(1, hash)
    statement.setString(2, user)
    statement.executeUpdate()
	  savePasswordChangeReport(user, hash).apply(connection) //really???
  }
```

Native `compose` and `andThen` functions can't really help here.
The fact that monadic chaining is our goal had been spoiler already. Let's get there step by step.

First of, we need a wrapper for a function:
```scala
case class DB[A](g: Connection => A):
  def apply(c: Connection): A = g(c)
```
To turn it into monad `map` and `flatMap` are required.

#### structure
```scala
case class DB[A](g: Connection => A):
  def apply(c: Connection): A = g(c)

  def map[B](f: A => B): DB[B] = DB(connection => {
    val valueA: A = g(connection)
    val valueB: B = f(valueA)
    valueB
  })

  def flatMap[B](f: A => DB[B]): DB[B] = DB(connection => {
    val valueA: A = g(connection)
    val dbB: DB[B] = f(valueA)
    val valueB: B = dbB.apply(connection)
    valueB
  })
```

Finally, `pure` will be helpful to wrap evaluated values (which one don't require connection):
```scala
object DB:
  def pure[A](a: A): DB[A] = DB(_ => a)
```

#### result
Now we can finally connect things together.

```scala
def getHash(user: String): DB[String] = ???
def savePasswordChangeReport(user: String): DB[Unit] = ???
def saveHash(user: String, hash: String): DB[Unit] = ???

def changePasswordIfSecretAreEqual(user: String, oldHash: String, oldSecretDb: String, newHash: String): DB[Boolean] =
  if (oldHash == oldSecretDb) {
    for {
      _ <- saveHash(user, newHash)
      _ <- savePasswordChangeReport(user)
    } yield true
  }
  else DB.pure(false)

def changePassword(user: String, oldSecret: String, newSecret: String): DB[Boolean] =
  for {
    oldSecretDb <- getHash(user)
    isEqual <- changePasswordIfSecretAreEqual(user, oldSecret, oldSecretDb, newSecret)
  } yield isEqual
```

#### Reader is just a function
With that path, we can return to the statement about my confusion. I said I thought Reader is just a function. After a while, I was unconsciously correct. Some additions are required though.

Cats treat functions as applicative functors.
So we can try to return our definitions into single-argument functions:

```scala
def getHash(user: String): Connection => String = _ => "old hash"
def savePasswordChangeReport(user: String): Connection => Unit = println
def saveHash(user: String, hash: String): Connection => Unit = println
```

To get `map`, `flatMap`, and `pure` we need FlatMap and Applicative instances.
```scala
import cats.syntax.flatMap._
import cats.syntax.applicative._
```

With those guys in place we can compose functions in the same way as readers.
```scala
def changePasswordIfSecretAreEqual(user: String, oldHash: String, oldSecretDb: String, newHash: String): Connection => Boolean =
  if (oldHash == oldSecretDb) {
    for {
      _ <- saveHash(user, newHash)
      _ <- savePasswordChangeReport(user)
    } yield true
  }
  else false.pure

def changePassword(user: String, oldSecret: String, newSecret: String): Connection => Boolean =
  for {
    oldSecretDb <- getHash(user)
    isEqual <- changePasswordIfSecretAreEqual(user, oldSecret, oldSecretDb, newSecret)
  } yield isEqual
```

We can try it out.
```scala
type Connection = String

changePassword("user", "old hash", "new hash")("connection")

//connection
//connection
//val res0: Boolean = true
```
Our functions are running, printing "connections", and returning comparison results.

#### conclusion
We still far away from motivation and we definetely hadn't been convinced to put Reader as return type to all functions of our traits. Moreover, it looks like alias of function.
Reader monad addresses very particular goal of explicit declaration which dependency should be provided to assemble things together. We leverage some portion of composability, nevertheless bringing pure reader monads "as is" is just way to fight with monad transformers and appropriate boilerplate (what we all love to do).
