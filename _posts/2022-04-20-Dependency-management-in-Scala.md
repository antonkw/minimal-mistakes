---
title: "Dependency management in Scala"
permalink: /applications/architecture/
# header:
#   image: /assets/images/writerjpg.jpg
excerpt: "Application dependency management in Scala: DI, ZIO and Monad Transformers"
categories:
  - applications
tags:
  - scala
  - scala cats
  - cats reader
  - scala reader
  - reader monad
  - cats mtl
  - scala tofu
  - scala architecture
  - scala monad transformers
  - scala applications architecture
  - scala app architecture
  - functional programming
classes: wide
---

That note pretends to be an overview of dependency management in Scala applications. 


## Motivation
My incentives to elaborate on this are messy a bit.
On the one hand, I initially attempted to get the motivation of ZIO ZLayer. That triggered the desire to keep in mind a common overview of things that people have used in Scala for years. ZIO has a separate “channel” to handle dependencies. That channel seemed over-engineered a little to me. [Part 4 - ZIO[Env, _, _]](https://www.youtube.com/watch?v=e1kIjiWHVhE&list=PLJGDHERh23x-_ammk-n2XuZWhoRVB-wAF&index=4) by [DevInsideYou](https://www.youtube.com/channel/UCSBUwLT9zXhUalKfJrc2q2A) is the brilliant intro, highly recommend it in case you need to get started with ZIO. 
On the other hand, disagreement with the ZIO posed questions regarding generally accepted practices. The status quo in terms of Tagless Final applications is not canonical FP. There is a mix of classes, constructors, and implicits. Evolution washed out different solutions. It is better to understand which steps we have previously made to reason about proceeding with the following steps.
I hope such a note could be useful for the community too since I didn't find a similar kind of aggregation. 

## Agenda

The plan I have for the note is the following.
1. Dead patterns. 
    1. DI frameworks.
    2. Cake pattern.
2. Pure ideas. 
    1. Dependency rejection.
    2. Dependencies as function parameters.
    3. The notion of the context for a function.
3. Tooling and approaches. Reader and friends.
    1. Reader monad.
    2. Dependency as API detail.
    3. Making unique contexts is not simple.
    4. ZIO ZLayer.
    5. Monad transformers (ToFu).

## Dead patterns
[Dependency Injection in Functional Programming by Gabriel Volpe](https://gist.github.com/gvolpe/1454db0ed9476ed0189dcc016fd758aa) and [Cake or Guice? : r/scala - Reddit](https://www.reddit.com/r/scala/comments/cnimu0/cake_or_guice) are colorful pieces of evidence of the fact that the community is aligned regarding using full-featured DI frameworks. 
There are some "outsiders" like [MacWire](https://github.com/softwaremill/macwire) which are not in the category of Guice/Spring-like frameworks. 
Some popularity of Guice was predictable as well as the following abandoning.
Scala survived Scala-as-better-Java misinterpretation times, and teams were looking for familiar and ready-to-use solutions. Guice was serving such a demand.
The same flow of reasoning explains why there is no space for Guice today. Talks around somehow mention referential transparency, the importance of careful compile-time verifications, etc. Tooling that does runtime reflections (for things we can prove at compile-time) is destined to have minimal support.
The cake pattern is not a matter of discussion too. [Cake antipattern (by kubuszok)](https://kubuszok.com/2018/cake-antipattern/)is an excellent note to get more argumentation.

## Pure ideas
### Dependency rejection
Mark Seeman carefully [explained](https://blog.ploeh.dk/2017/02/02/dependency-rejection/) the principle itself.
It is slightly out of topic since the notion doesn't help organize dependencies. It just prompts us to get rid of dependencies. Notwithstanding, the idea recalls the importance of keeping any side effects as close to the edge as possible.
What does that mean?
First, we should care about decoupling business logic and IO. [DevInsideYou](https://www.youtube.com/c/DevInsideYou/videos)[has example](https://www.youtube.com/watch?v=HE3qCrLWgYw&list=PLJGDHERh23x-3_T3Dua6Fwp4KlG0J25DI&index=6) that pushes the idea to the max by keeping all persistence in the isolated module. However, in practice, business apps are full of interactions with the globe. Hence, there are not many things to isolate. 
Fortunately, I have an excellent example. I was lucky to work in a team that improves an algorithm that matches the DNAs of patients and potential donors. And what the team had made perfectly is the isolation of core functions like `(DNA, DNA) => Score` in a separate library. It doesn't matter which business feature will trigger another portion of CRUDs, new caches, and databases. Pure functions represent algorithms and live in their pure world; hence, people could develop and test core logic without thinking about dependencies. That example scales well; it sounds natural to recommend that everybody move pure functions into a separate layer. We use the dependency rejection principle when we decide to postpone a meeting with a real impure world. 
It is pretty easy to come up with naïve formalization. Once we have effect tracking, we can switch to terms of signatures. 
That's an appropriate moment to cite the original note's conclusion:
>Dependencies are, by nature, impure. They're either non-deterministic, have side-effects, or both. Pure functions can't call impure functions (because that would make them impure as well), so pure functions can't have dependencies.

It ends up with an additional easy-to-understand declaration that sounds pretentious, though. We implement the idea of dependency rejection when we isolate processing steps inside non-effectful (`A = > B` instead of `A => IO[B]`) functions. The restrictions are clear. In terms of application architecture, we can master the principle mainly within the confines of pure functions. Specifically here, theoretical assumptions crash into the rocks; we need to roll up our sleeves and venture into the murky water of IO and app dependencies.

### Dependency parametrization
That means passing dependencies as arguments to functions.
In Scala, we have more options than mixing real arguments with dependencies.

##### Partially applied functions
[partially-applied functions](https://alvinalexander.com/scala/fp-book/partially-applied-functions-currying-in-scala/)is a straightforward way to draw the line between app dependencies and actual arguments.

```scala
object UserService:
  def register(repo: UserRepository, emailClient: EMailClient)(user: User): F[UserId] 
```
That option is canonical FP. So we have just a function that does some manipulations with arguments and returns a result.
Partial application and currying are helping to make re-usage more or less comfortable: 
`val register: User => F[UserId] = UserService.register(userRepository, emailClient)`
The major drawback is obvious. Anybody who wants to call the function still ought to manage dependencies and pass them to every function. 

##### implicit arguments
Next step is passing dependencies as implicit arguments.
```scala
object UserService:
  def register(user: User)(using repo: UserRepository, emailClient: EMailClient): F[UserId] 
```
Now we're free not to specify dependencies as parameters. Nevertheless, any client ought to "own" instances to provide them as implicits. So, we add some sugar to pass things less verbose, but it is still not a receipt for the problem.

### Unique contexts for each function
The idea lies just beneath the surface.
Once we dislike managing dependencies one by one, it sounds sensible to aggregate them.

Let us re-use an example:
```scala
object UserService:
  def register(user: User)(using repo: UserRepository, emailClient: EMailClient): F[UserId] 
```

Let's pack together all service classes.
```scala
case class RegisterContext(repo: UserRepository, emailClient: EMailClient)

object UserService:
  def register(user: User)(using ctx: RegisterContext): F[UserId] 
  def deregister(user: User)(using ctx: RegisterContext): F[UserId] 
```

While it looks good in terms of individual function, it makes little sense further. For example, let's we need to de-register the user. And the process implies having another third party like `UserDeregistrationAuditLog`. 

Now we have another context.
```scala
case class DeregisterContext(
  repo: UserRepository, 
  emailClient: EMailClient,
  userLog: UserDeregistrationAuditLog
)
```
Proceeding with that idea, we meet two poor options:
- An infinite amount of unique contexts.
- A limited amount of reusable contexts.

The first approach brings the cumbersome task of keeping all contexts neat and ordered.
The second approach inevitably shares more access components than necessary.

What can we summarize here? Gathered contexts are beneficial only for handling things necessary by **all** components of an application (logger, tracing, etc.). "Effectful context" under the fourth chapter of [Functional event-driven architecture](https://leanpub.com/feda) is a must-read for that topic.

The vital question that we can pose for the next iteration. Can we use some shared context but thrust only some parts of it to consumers?

## Tooling
#### Reader monad!
Adam Warski in [Reader & Constructor-based Dependency Injection - friend or foe?](https://softwaremill.com/reader-monad-constructor-dependency-injection-friend-or-foe/) states:
> Reader Monad can be viewed as a basic way to **track effects** in our code, by explicitly stating that certain dependencies can be used down in the call chain.

An important statement that I want to unfold.
On one side, it is easy to provide an example where Reader advantageously emphasizes the nature of underlying interactions.
An example from Adam's post is excellent:
```scala
object UserNotifier:
  def notify(user: User, about: String): Reader[EmailServer, IO[Unit]]
```
We can quickly ascertain how an app delivers notifications to the user.
Moreover, `EmailServer` could be abstract. Hence, we can enrich API without showing up on implementation details.
Nonetheless, there are multiple predicaments we can meet in real applications.

###### effects and application services are different abstractions
Once we're talking about application architecture, we need to think about arranging layers.
For instance, let's try to use Reader to manage the "program" level. 
[CheckoutProgram](https://github.com/gvolpe/pfps-shopping-cart/blob/master/modules/core/src/main/scala/shop/programs/checkout.scala) from [pfps-shopping-cart](https://github.com/gvolpe/pfps-shopping-cart)looks quite standard; there are some clients, a couple of services, and configuration.
We can pack it into a large tuple and check if declared transparency is in place.
```scala
object CheckoutProgram:
  def checkout(userId: UserId, card: Card): 
    Reader[
      (PaymentClient[IO], ShoppingCart[IO], Orders[IO], RetryPolicy[IO]), 
      IO[OrderId]
    ] = ???
```
We stopped observing any effect-related transparency at the program level. Instead, we thrust ourselves cumbersome context that we ought to maintain further.
To enable **effects** tracking, we are potentially required to build a context that unleashes underlying effects and particular functions.
It should look like this:

```scala
class CheckoutEnv[
  A[_]: GenUUID: MonadThrow, 
  B[_]: Sync, 
  C[_]: JsonDecoder: BracketThrow
](
  createOrder: (UserId, PaymentId, List[CartItem], Money) => A[OrderId]
  findCard: UserId => B[CartTotal],
  processPayment: Payment => C[PaymentId]
)
```

Here we are, all actions are in place and everybody can assume what implementation is going to do. Nowwithstanding, price is too high. Every function now require cooking its own environment (like `CheckoutEnv`).

Therefore, it is fair to rephrase the original statement. What we can attempt to track is not effects (at least in terms of vocabulary we imply with Tagless Final and effect systems). What makes sense is that Reader helps bring the notion of context (or environment) necessary for computation.

With such phrasing, we end up with an open ending. Bringing the context doesn't sound beneficial in itself. But it paves the way for different re-usages.

Questions that I want to pose and postpone.
1. How often do we need to show dependencies as part of interfaces? What can we leverage by doing that? Are there *consequences*?
2. Are there means to specify necessary parts precisely without building unique contexts everywhere?

#### Dependency as API detail
The lowest common denominator of many discussions of "reader or constructor" is the argument that the reader exposes details required to build *instances of abstractions*.
At that moment, I don't want to facilitate counter-arguments. So instead, I suggest just inverting it. Can we say "classes with constructors don't allow to bring context-details as part of abstractions"? Sounds not cool and straightforward. I hope a tiny example will be helpful.
Let us have a cache.
```scala
trait Cache[K, V]:
  def get(k: K): V
  def put(k: K, v: V): Unit
```
And now, we need to enrich semantics by saying "that cache use load function to get non-presented value."
Constructor-based mindset prompts just to interpret it as an implementation detail.
```scala
object CacheLive {
  def make[K, V](loadFunction: K => IO[V]): Cache[K,V]
}
```
But can we bring the notion of load function as part of semantics? We're definitely in trouble. There is no single receipt to say "that cache use load function to acquire non-presented value."

But anybody who will take a look at API will be inclined to think that sole way to get value is to put it there first. We're just misleading people!
Moving `loadFunction` to the signature (of `get` ) give hint that we should manage those load functions. But we actually should emphasize that load function is kind of configuration of our *context*. Word "context" should remind us about implicits. In fact, implicit argument is good option. The problem I see is that we all used to not think about implicits like about part of abstraction. Loggers, encoders, all stuff like that are typically "technical support". As a result, anything passed as implicit could be perceived as annoying stuff (which hopefull will be automatically resolved by some magic import).
Hence, the better option is to say that cache explicitly requires a fixed load function. And Reader is just the right tool to denote that fact.

```scala
trait ValueLoader[K, V]:  
  def load(k: K): V  
  
trait Cache[K, V]:  
  def get(k: K): Reader[ValueLoader[K, V], V]  
  def put(k: K, v: V): Unit
```

Now we precisely got what we implied. 
<details>
<summary>But the load function should return IO!</summary>

It is easy to notice that returning to the version with IO will make the signature more difficult.

```scala
trait ValueLoader[F[_], K, V]:  
  def load(k: K): F[V]  
  
trait Cache[F[_], K, V]:  
  def get(k: K): Kleisli[F, ValueLoader[F, K, V], V]  
  def put(k: K, v: V): Unit  
  
class CacheLive[K, V] extends Cache[IO, K, V]:  
  override def get(k: K): Kleisli[IO, ValueLoader[IO, K, V], V] = Kleisli(loader => loader.load(k))  
  override def put(k: K, v: V): Unit = ???  
```
We'll try to that signature later.
</details>
Important thought here is a conscious movement toward a better *explanation of what is required to construct implementation.*
Striving for encapsulation can nudge us to hide things instead of making a reasonable declaration.

We meet an open ending again. The point here is understanding that sometimes we might want to lift dependencies to API declaration.

#### Large context without exposure of internals
We can now try to tackle the "context trouble" we set out previously.
We can't list app services straightforwardly, even without precise details and transparency concerns. Previously, we attempted to pack services (like `PaymentClient`, `ShoppingCart`) into Reader.

We stated that we have two contrary alternatives:
- **Cook a unique environment for each function.** It barely works; formerly, we strive for better dependency management. Creating specific contexts with all necessary dependencies (and we should pass everything through the layers) could not be part of such a plan.
- **Re-use one context everywhere.** At that point, we're entirely missing the transparency point. Readers want an environment that contains everything literally. With that approach, we can't even observe a dependency graph in any form since everybody can use any part of the context.

There is still a tempting idea to union benefits of both somehow. It would be great to have some context we had built once. Herewith, we also need some sophisticated means not to specify a direct dependency on the context. It would be great to force an abstract context to provide any necessary parts. That's how we might leverage the strength of both approaches.

The different question is how to tackle the task.

Let we have couple of traits:
```scala
trait PaymentClient:  
  def processPayment(p: Payment): Id[TransactionId]  
  
trait ShoppingCart:  
  def findCard(userId: UserId): Id[CartTotal]
```

And we want to define the signature for the function that will use both traits as part of Reader's context:

```scala
type CheckoutContext = ???
  
def checkout(userId: UserId, payment: Payment): Reader[CheckoutContext, TransactionId]
```

On client side it is tempting to perceive CheckoutContext as implementation of all necessary traits. 
```scala
type CheckoutContext = PaymentClient & ShoppingCart  
  
def checkout(userId: UserId, payment: Payment): Reader[CheckoutContext, TransactionId] =  
  Reader(ctx => ctx.findCard(userId) *> ctx.processPayment(payment))
```
The sole *predicament* is implementation of `CheckoutContext`. In fact, we want to have some environmental instance that implements all traits of application services. 
For sure, we can roll up our sleeves and just dig in. But implementing all traits sounds extremely miserable and reminds both "god object" and "cake" anti-patterns. 


###### ZIO Has and ZLayer
ZIO implements that stated idea and provides convenient composition building blocks.
[Has](https://zio.dev/version-1.x/datatypes/contextual/has/) trait brings necessary magic that allows describing expectations from the environment. The reader channel of ZIO enforces boundaries, and the user ought to provide an environment that supplies required parts. 

With ZIO that signature would look like:
```scala
type CheckoutContext = Has[PaymentClient] & Has[ShoppingCart] 
  
def checkout(userId: UserId, payment: Payment): UIO[CheckoutContext, TransactionId]
```
As you can see, `Has` handles some portion of magic. ZIO provides nice tooling to transform that "implementation of all traits" problem into an efficient graph-like composition. [ZLayer](https://zio.dev/version-1.x/datatypes/contextual/zlayer)  helps with cooking such an environment. [DevInsideYou ZIO playlist](https://www.youtube.com/watch?v=e1kIjiWHVhE&list=PLJGDHERh23x-_ammk-n2XuZWhoRVB-wAF&index=4&ab_channel=DevInsideYou) is the right source to familiarize with details.
Long story short, ZIO promotes syntax to finally have typing composition like:
```scala
val checkout: ZLayer[Any, Nothing, CheckoutContext] = paymentClient ++ shoppingCart
```
Don't be confused by those Any and Nothing placeholders. The purpose of the snippet is to show that it is possible to combine two components somehow and receive the summarized CheckoutContext type.
It is essential to understand that mentioned tooling targets application-level dependency management; interaction with the external world is a different topic. It is still recommended not to entwine API of libraries or external modules with environmental dependencies.
###### Monad transformers and friends
Previously, we tackled the cache with the value loader function.
It looked awkward:
```scala
trait Cache[K, V]:  
  def get(k: K): Reader[ValueLoader[K, V], V]  
  def put(k: K, v: V): Unit
```

We also need to add effects.
```scala
trait ValueLoader[F[_], K, V]:  
  def load(k: K): F[V]  
  
trait Cache[F[_], K, V]:  
  def get(k: K): Kleisli[F, ValueLoader[F, K, V], V]  
  def put(k: K, v: V): F[Unit]
```

Now it is tough to argue why we do not have a single `K => F[V]` signature for `get`. We can easily inject the load function and not mention it in API:
```scala
object CacheLive {
  def make[F[_], K, V](loadFunction: K => F[V]): Cache[F, K, V] = ???
}
```

But can we somehow *enrich* the signature without being too lengthy?

We could state the formal task as altering `Kleisli[F, ValueLoader[F, K, V], V]` into something different. Do we have an option to replace it with `F[V]` and move all details into the description of the context bounds? 

First, we can't stay with traits. `trait Cache[F[_]: Monad]`  doesn't work due to "Traits cannot have type parameters with context bounds." Hence, we need to step down to the level of classes.

What we used to do once we had any predicaments with some particular monad? Correct! We're looking for the proper signature through transformers APIs.
"Native" tooling for cats is [Cats MTL](https://github.com/typelevel/cats-mtl). Let me be ingenious here and consider [ToFu](https://docs.tofu.tf/)instead. In fact, I just like provided syntax and *wording*. 

After some playing around, `WithContext`-version could look in the following way:
```scala
abstract class LoadedCache[F[_]: FlatMap: WithContext[*[*], ValueLoader[F, K, V]], K, V] extends Cache[F, K, V]: 
  def get(k: K): F[V]  
```

`WithContext[*[*], ValueLoader[F, K, V]]` literally means obligation to provide value loaded. The user, in its turn, receives the capability to ask for a value loader.
Implementation itself will *ask* for context: `def get(k: K): F[V] = F.askF(_.load(k))`.

The only missing part is the feeding instance of the loaded cache.
```scala
implicit val ctx: WithContext[IO, ValueLoader[IO, String, String]] =  
  WithContext.const[IO, ValueLoader[IO, String, String]]((k: String) => IO.pure(k.toUpperCase))
```

Line above is responsible for providing value loader. 

Final usage is extremely simple. Everything looks as conversation with simple `K => F[V]` function.
```scala
val cache = new LoadFCacheImpl[IO, String, String]  
val example: IO[String] = cache.get("s")
```

What is important, lack of necessary context "companions" will be carefully explained.
```
could not find implicit value for evidence parameter of type WithContext[IO,ValueLoader[IO,String,String]]
```
So, instance of `WithContext` is glue we need to wire stated dependency and actual value.

We addressed our initial problem but with workarounds:
1. We needed to move to the class-level to enrich the semantics of API; replacing traits with classes (with less abstract boundaries) can bring a lot of mess.
2. Decoupling context from its consumers is not trivial. We bring more tooling and additional layer of abstraction. It's a high price.

So where does that leave us? "What Did It Cost? Everything." Modern Scala is moving towards being friendly and easily adoptable. Solving problems in canonical way is fun but could do a great disservice too.

#### Summary
Long story short, Scala ecosystem has couple of main ways to orginize dependencies.
1. Canonical Typelevel stack usually implies arrangement with constructors. Also, classes that [handle some subset of dependencies](https://github.com/gvolpe/pfps-shopping-cart/tree/second-edition/modules/core/src/main/scala/shop/modules) help keep neat even large graphs. Finally, environmental services are free to be passed implicitly.
2. ZIO ecosystem promotes idea of having separate Reader channel. ZIO also provides good toolset to make usage as simple as possible. That means users have standardized practices to not fight with problems of pure Reader monad.

Those are generic trends, but things are not carved in stone.
Lightweight DI frameworks could still help wire dependencies together; the next generation does it gently without taking on too much.
Libraries like Cats MTL and ToFu help do non-trivial tricks in a couple of lines.

That means there is no standardized way to undertake the task. Nevertheless, all approaches have a focus to be simple for reasoning. That's good news. There is plenty of tooling, and we're free to pick the best suitable options.
