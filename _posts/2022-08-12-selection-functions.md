---
title: "Selection functions is interesting notion with tricky implementation"
permalink: /scala/selection-functions/
# header:
#   image: /assets/images/writerjpg.jpg
excerpt: "Selection functions is topic you can find mostly in research papers. My intention is helping with first steps. The notion itself is simple but implementation shows unobvious patterns."
categories:
  - scala
tags:
  - scala
  - selection functions
  - selection monad
  - functional programming
  - selection functions scala
classes: wide
---
"Sequential Games and Optimal Strategies" by Martın Escardo and Paulo Oliva has the best possible introduction:
> Life is the sum of all your choices, so said Albert Camus. But what does “choice” mean? One could say that to choose is to select one element x out of a set X of possible candidates.

The note will be devoted to selection functions that are all about choices from some sets of options.  
Selection functions are not something we can leverage in daily programming.
Notwithstanding, playing around with selection functions was fun for me. I stuck a couple of times because of approaches that were not obvious to me. And adopting new thinking patterns is something why we all love programming.

I hope you'll enjoy some pieces too.

There is a decent amount of good papers on the topic.

Few to name:
- [Sequential Games and Optimal Strategies](https://www.eecs.qmul.ac.uk/~pbo/papers/paper032.pdf) by Martín Escardó and Paulo Oliva.
- [Finding optimal strategies in sequential games with the novel selection monad](https://arxiv.org/abs/2105.12514) by Johannes Hartmann.
- [What Sequential Games, the Tychonoff Theorem and the Double-Negation Shift have in Common](https://research.birmingham.ac.uk/en/publications/what-sequential-games-the-tychonoff-theorem-and-the-double-negati) by Martín Escardó and Paulo Oliva.

What I attempt to do is to make those papers a little bit closer to Scala practitioners.

## Selection function
The signature of selection functions is simple:
```scala
type J[R, A] = (A => R) => A
```
The main notion there is that function has a set of objects of type `A` and provides a way to select one of those objects.
Formally, the selection function embodies the collection of objects, judges them according to provided criteria, and returns the best option.
What is unusual here is that the collection is not a parameter or a part of the context. The selection function hides it under the hood.

Let's come up with some toy examples.
The simplest case is:
- Having a list of entities.
- Maximizing some property.

So, for abstract  `Entity` and `Property` we'll need a collection `List[Entity]` itself and ordering `Order[Property]`.
```scala
def maxWith[Entity, Property](  
  entities: List[Entity],
  order: cats.Order[Property]  
): J[Property, Entity] =  
  (evaluate: Entity => Property) => 
    entities.maxBy(evaluate)(order.toOrdering)
```

Now we can pick a brand-new BMW!
```scala
case class BMW(
  model: String, 
  power: hp, 
  msrp: USD
)  
  
val bmws =  
  List(  
    BMW("230i", 255, 38_395),  
    BMW("330e", 288, 46_295),  
    BMW("M550i", 523, 60_945)  
  )

def bmwSelection[Property](
  implicit order: Order[Property]
): J[Property, BMW] = 
  maxWith[BMW, Property](bmws, order)

```

The particular selection function is going to look like this:
```scala
val bmwSelectionByHp: J[hp, BMW] = bmwSelection[hp]
```

The last step is to let the function know which property to use.
```scala
val powerfulBmw: BMW = 
  bmwSelectionByHp(_.power) /// BMW(M550i,523,60945)
```

## Quantifier functions
There are also quantifier functions that provide not a selected object but a property that "causes" a choice.
```scala
type K[R, A] = (A => R) => R
```

For sure, we can derive quantifiers from selection functions.

```scala
def overline[R, A]: J[R, A] => K[R, A] =  
  (select: J[R, A]) =>  
    (eval: A => R) => {  
      val selected: A  = select(eval)  
      val evaluated: R = eval.apply(selected)  
      evaluated  
    }
```

```scala
val maxPowerOfBmw = overline(bmwSelectionByHp)(_.power) // 523
```

# Combining two selections
Things become much less obvious once we try to "glue" a couple of functions together.
Selection functions should help with choices. And choices should be made in combination with other choices. Let's start for pair of selections.

## Signature of the pairing function
First of all, we need to agree on the signature.
Let us have two functions: `J[R1, A]` and `J[R2, B]`.
We want to combine *choices*. That means that we expect that choices could have different nature. So, distinguishing `A` and `B` is desired flexibility.
And what about typing for "truth values"? We can foresee that values to judge the `A`/`B`-combination is our "pairing point."
We might want to have the same return type of evaluation function. In that case, pairing `J[R, A]` and `J[R, B]` should lead to a selection like `J[R, (A, B)]`.  We had individual `A => R` and `B => R`, and to judge pair we'll need `(A, B) => R`.
With separate `R`s, there will be no "contact points." An exploded signature is not a problem by itself. The missed piece, in that case, is a lack of notion of common judgment in terms of a particular combination.

Well, we have the signature.
```scala
def pair[R, A, B]: J[R, A] => J[R, B] => J[R, (A, B)]
```

## Enabling predicates
It is better to explore a non-obvious usage case to understand `pair` implementation.
Obviously, the selection function does full iteration over all underlying elements. It applies `X => R` transformation to choose the best available option.
That means we can tune our evaluation function to search for an exact match. It would cover the demand of the "brute-forcing" selection function. We turn the evaluation function into the strict predicate.
```scala
val bmwPredicateSelection = bmwSelection[Boolean]  
val matched = bmwPredicateSelection(_.model == "M550i") // BMW(M550i,523,60945)
```
We don't need to do much to enable predicates. `Boolean` is a completely valid "truth value" type. Standard ordering makes `true` values larger than `false` ones.

## Implementation of the pairing function
Well, it is time to return to the combination.
We have separated selections for `A` and `B`. And also evaluation for a tuple of them.
```scala
def pair[R, A, B]: J[R, A] => J[R, B] => J[R, (A, B)] =  
  (selectA: J[R, A]) =>  
    (selectB: J[R, B]) =>  
      (evaluateAB: ((A, B)) => R) => ???
```

The main trick here is to realize that there are **no independent judgments** yet. So, we need to build tuples and judge them.
Once we construct the evaluation function (argument for `selectA`) it is necessary to also select the best possible instance of `B`.
```scala
def pickUpB(candidateA: A): B => R = ???

val a: A = 
  selectA(
    candidateA => 
      //key -> value is sugar to create tuple2
      evaluateAB(candidateA -> selectB(pickUpB(candidateA)))
    )
  )
```

And "the best possible instance of B" means the best combination with a preliminary instance of A.
```scala
def pickUpB(candidateA: A): B => R =
  (candidateB: B) => 
    evaluateAB((candidateA, candidateB))
```

The final touch is the re-selection of `B` after having selected `A`.
```scala
def pair[R, A, B]: J[R, A] => J[R, B] => J[R, (A, B)] =  
  (selectA: J[R, A]) =>  
    (selectB: J[R, B]) =>  
      (evaluateAB: ((A, B)) => R) => {  
        def pickUpB(candidateA: A): B => R =  
          (candidateB: B) => evaluateAB((candidateA, candidateB))  
          
        val a: A = 
          selectA(
            candidateA => 
              evaluateAB(
                candidateA -> selectB(pickUpB(candidateA)
              )
            )
          )
           
        val b: B = 
          selectB(candidateB => evaluateAB(a, candidateB))  
        
        (a, b)  
      }
```

Now we can play around with predicates for combinations.
Let's brute-force a two-char password with the help of two selection functions that formerly worked independently.
```scala
type Password = (Int, Char)  
  
val passwordCriteria: Password => Boolean = 
  { case (a, b) => a == 7 && b == 'p' }

def bruteforceInt[T](implicit order: Order[T]): J[T, Int] = 
  maxWith(order, (1 to 9).toList)  
def bruteforceChar[T](implicit order: Order[T]): J[T, Char] = 
  maxWith(order, ('a' to 'z').toList)  
  
val password = 
  pair[Boolean, Int, Char]
    .apply(bruteforceInt)(bruteforceChar)(passwordCriteria)

println(s"Secret: $password") //Secret: (7,p)
```


# More functions combined together
Now we are closer to useful applications of selection functions.

The core notion here is representing our problems (like playing games or graph searching) as a series of choices.
Previously we combined two selections into one. In the same manner, we can transform a sequence of `A`-selections into a single selection that will select the best option from the sequence of potential `A`-objects: `List[J[R, A]] => J[R, List[A]]`.

Pay attention that at this step, we need the function `List[A] => R` that will judge the whole sequence.

The approach is the same. We just iteratively produce selections with all possible candidates.

```scala
def product[R, A]: List[J[R, A]] => J[R, List[A]] =  
  (functions: List[J[R, A]]) =>  
    (listEval: List[A] => R) =>  
      functions match {  
        case evalFun :: restEvalFunctions =>  
          val a: A =  
            evalFun((candidate: A) =>  
              listEval(  
                candidate ::  
                  product  
                    .apply(restEvalFunctions)  
                    .apply(  
                      restCandidates =>   
                        listEval(candidate :: restCandidates)  
                    )  
              )  
            )  
              
          val as: List[A] =   
            product(restEvalFunctions)(rest => listEval(a :: rest))  
              
          a :: as  
  
        case Nil => Nil  
      }
```

The code there is highly non-obvious.
1. On a high level, we iterate over the list of selection functions.
2. For every element, we consider another one function (standard `head :: tail` pattern matching).
3. For every element, we calculate the candidate `val a: A` in terms of both current `evalFun` and the rest of the functions (`restEvalFunctions`) together. Candidate `(candidate: A)` is judged as part of the whole solution (list of `A`-objects) by `listEval`.
4. Recursively, we re-evaluate logic and calculate the "tail."  We already know the best candidate for the current iteration and pass it as part of the known part of the solution: `listEval(a :: rest)`.


<details markdown="block">  
<summary markdown="span">Let's add more logging and counters</summary>  


```scala
var n = 0  
def productWLogs[R, A]: List[J[R, A]] => J[R, List[A]] =  
  (functions: List[J[R, A]]) => {  
    n = n + 1  
    (evalList: List[A] => R) =>  
      functions match {  
        case e :: es =>  
          val a: A =  
            e { (candidate: A) =>  
              evalList  
                .compose { (list: List[A]) =>  
                  println(s"1. Considering $list for $candidate on ${n}th iteration"); list  
                }  
                .apply(  
                  candidate :: productWLogs  
                    .apply(es)  
                    .apply(arg =>  
                      evalList  
                        .compose { (list: List[A]) =>  
                          println(s"2. Considering $list for $candidate on ${n}th iteration");  
                          list  
                        }  
                        .apply(candidate :: arg)  
                    )  
                )  
            }  
  
          val as: List[A] =  
            productWLogs(es)(arg =>  
              evalList  
                .compose { (list: List[A]) =>  
                  println(s"3. Considering $list for $a on ${n}th iteration");  
                  list  
                }  
                .apply(a :: arg)  
            )  
          a :: as  
  
        case Nil => Nil  
      }  
  }
```
</details>

The function does even more than n² iterations! 757 is the actual counter value, while 26² is equal to 676.

So, solutions are being generated multiple times. We have no "caching" mechanism. We re-generate the same combinations to pick up the best solutions during sequential iterations.

There are some feasible optimisations but I suggest to switch to the completely opposite approach. We can just be as greedy as possible. But we need a lightly different evaluation function to tackle partial solutions.

# Greedy product
To reduce complexity, we can use a greedy approach.
That means just picking the best solution for each step. And these per-step solutions are not full-featured combinations but the best options for particular iterations.

```scala
def greedyProduct[R, A]: List[J[R, A]] => J[R, List[A]] =  
  (selections: List[J[R, A]]) => { (evalList: List[A] => R) =>  
    selections match {  
      case eval :: es =>  
        val candidate: A  = eval((candidate: A) => evalList(candidate :: List()))  
        val rest: List[A] = greedyProduct(es)(arg => evalList(candidate :: arg))  
  
        candidate :: rest  
  
      case Nil => Nil  
    }  
  }
```

So, we use the `evalList` function to evaluate the singleton list.
```scala
val candidate: A = 
  eval(
    (candidate: A) => 
      evalList(candidate :: List())
  )  
```

After it, we proceed with the rest of the functions.

To leverage the greedy approach, we need appropriate evaluation functions. Functions should be able to cope with partial solutions.
In the case of passwords, we can measure how many characters are equal to the target.
```scala
val stringEquality: String => (List[Char] => Int) =  
  target => 
    guessed => 
      target
        .zip(guessed.mkString)
        .count { case (c1, c2) => c1 == c2 }
```

```scala
val result = // List(p, a, s, s, w, o, r, d)  
  greedyProduct(  
    List.fill(8)(bruteforceChar[Int])  
  )(stringEquality("password"))
```

Please note that greedy algorithms, by their nature, can provide non-optimal solutions.

# Selection monad
In highly unobvious manner, selection functions form monad.

First, let's pack the function into convenient container.
```scala
case class Selection[R, A](e: J[R, A])
```

`flip` from Haskell will be useful.
```scala
def flip[A, B, C](f: A => B => C)(x: B)(y: A) = f(y)(x)
```

Well, `Monad` instance implements chaining of selections with changing type of entities with same `R` type (return type of evaluation functions).
```scala
implicit def makeMonad[R] = new Monad[Selection[R, *]] {  
    override def flatMap[A, B](fa: Selection[R, A])(f: A => Selection[R, B]): Selection[R, B] =  
      Selection((p: B => R) => f(fa.e(a => p(flip(f.andThen(_.e))(p)(a)))).e(p))  
  
    override def map[A, B](fa: Selection[R, A])(f: A => B): Selection[R, B] =  
      Selection((p: B => R) => f(fa.e(a => p(f(a)))))  
  
    override def pure[A](x: A): Selection[R, A] = Selection(_ => x)

    override def tailRecM[A, B](a: A)(f: A => Selection[R, Either[A, B]]): Selection[R, B] = ???  
}
```

As artificial example, I can suggest to pick up the most expensive BMW, and for rest of money look for EV car. We use same `USD` property to judge different entities. And we can take into account previously selected object.
```scala
case class EvCar(name: String, price: USD)  
val evCars: List[EvCar] = List(...)

val usdSelectionMonad: Monad[λ[a => Selection[USD, a]]] = makeMonad[USD]

def evSelectionWithLimit(limit: USD) =  
  Selection[USD, EvCar]((eval: EvCar => USD) =>  
    evCars.filter(_.price < limit).maxBy(eval)  
  )
  
val evCarSelection: Selection[USD, EvCar] = usdSelectionMonad.flatMap(bmwSelectionByPrice)((bmw: BMW) =>  
  evSelectionWithLimit(100_000 - bmw.msrp)  
)
```

# Summary
I attempted to build a short but meaningful introduction to selection functions.
Selection functions are the topic that is being actively researched.
And research papers can quickly dive into the topic. And they could imply that readers have some related background.
I intended to give practical guidance for the initial steps. I hope it can help to gain acknowledgment and not lose the narrative thread (in those papers).

# Resources
- [Sequential Games and Optimal Strategies](https://www.eecs.qmul.ac.uk/~pbo/papers/paper032.pdf) by Martín Escardó and Paulo Oliva. That is the entry point to many different related kinds of research.
- [Finding optimal strategies in sequential games with the novel selection monad](https://arxiv.org/abs/2105.12514) by Johannes Hartmann is 60 pages of careful explanations. The very basics end with implementations of Sudoku and Chess.
- [Algorithm Design with the Selection Monad](https://trendsfp.github.io/2022/tfp-abstracts.pdf) by Johannes Hartmann and Jeremy Gibbons is focused on algorithms.
- [The ubiquitous selection monad](https://www.cs.bham.ac.uk/~mhe/.talks/wessex2011/wessex2011.pdf) by Martín Escardó and Paulo Oliva.
- [What Sequential Games, the Tychonoff Theorem and the Double-Negation Shift have in Common](https://research.birmingham.ac.uk/en/publications/what-sequential-games-the-tychonoff-theorem-and-the-double-negati) by Martín Escardó and Paulo Oliva.
- [Selection functions and lenses](https://julesh.com/2021/03/30/selection-functions-and-lenses/) by Jules Hedges.