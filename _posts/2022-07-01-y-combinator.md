---
title: "Write Y Combinator to understand recursions better"
permalink: /scala/y-combinator/
# header:
#   image: /assets/images/writerjpg.jpg
excerpt: "Implementation of Y Combinator from scratch (in Scala). It helps to understand structure of recursions."
categories:
  - scala
tags:
  - scala
  - scala cats
  - cats recursions
  - tail recursion
  - cats tailRecM
  - functional programming
  - y combinator scala
classes: wide
---

# Introduction
Argumentation about the necessity to familiarize with fixed points lies between opposite stances:
- Fixed points topic is part lambda calculus; no real demand to even read about it.
- Fixed points are required to grasp recursive schemes. And recursive schemes are an essential topic on the FP agenda.

Both make sense. I don't think somebody can generally be stuck in a career  due to "non-knowing recursions." But still, everyday programming ends up with some recursions sometimes. And better acknowledgment makes life easier.

On my side, I found that I had a lack of naive feeling about what is the fixed point. Initially, I attempted to dive into the [matryoshka](https://github.com/precog/matryoshka)library. There was not a single attempt but approaching from time to time. And every time, I was scared by the number of prerequisites required to understand the basics.

One of such fundamental conceptions is the Y combinator. There is a vast amount of diverse materials on the topic. And still, I spent a lot of time gathering pieces together. I am sharing my path with the hope that somebody will be able to cut a path.

The agenda for today is:
- A little bit of theory to (not) understand what is a fixed-point combinator.
- Attempt to withdraw self-reference from the classic factorial definition.
- Writing Y Combinator step by step.
- Writing code you might want to use in real life (no fixed points, just `tailRecM` from cats).


# Theory
Further, I'll be focused on practical aspects. That means writing code as way to build naive understanding. Before doing that, let's take a look at formal definitions.

Wiki [says](https://en.wikipedia.org/wiki/Fixed_point_(mathematics)):
>**fixed point** (also fixpoint or invariant point) of a function is an element that is mapped to itself by the function. That is, **c** is a fixed point of a function ***f*** if ***c*** belongs to both the domain and the codomain of **f**, and **_f_(_c_) = _c_**.

For example, if _f_ is defined on the real numbers by
***f(x)=x<sup>2</sup>-3x+4***
then ***2*** is a fixed point of **_f_**, because ***f(2) = 2***.

The next brick is [Fixed-point combinator](https://en.wikipedia.org/wiki/Fixed-point_combinator)(or Y combinator):
>**fixed-point combinator** is a higher-order function that returns some fixed point of its argument function, if one exists.

And more well-formulated motivation could be found on [Rosetta Code page](https://rosettacode.org/wiki/Y_combinator):
>
In strict functional programming and the lambda calculus, functions (lambda expressions) don't have state and are only allowed to refer to arguments of enclosing functions.
This rules out the usual definition of a recursive function wherein a function is associated with the state of a variable and this variable's state is used in the body of the function.
The Y combinator is itself a stateless function that, when applied to another stateless function, returns a recursive version of the function.

## Resources
Also, I want to share a couple of resources. You can skip them but I still wanted to make it part of the initial familiarization. Some code transformations could be easier for understanding with those materials in mind.
- The basic understanding of Lambda Calculus is going to be useful today. There is an extremely good talk to quickly gain acknowledgment: [Lambda Calculus - Fundamentals of Lambda Calculus & Functional Programming in JavaScript](https://www.youtube.com/watch?v=3VQ382QG-y4).
- [Essentials: Functional Programming's Y Combinator - Computerphile](https://www.youtube.com/watch?v=9T8A89jgeTI) - lightweight explanation for those who like the academic way of definitions. For the rest of us, it could be just starting point of building a more solid perception.

# Let's code!
Scala nudges us to start from the end. We have the support of recursions and we can just use it. Whether we like it or not, we always refer to our experience. So, I suggest to not deriving anything from scratch, but making a function *less recursive*.  We used to use recursive definitions and now we have a chance to understand the mechanism better.

## recursion that we (don't) know
No surprise, the standard candidate to cope with is factorial:
```scala
def factorial(x: Int): Int =
  if (x == 0) 1 else x * factorial(x - 1)
```

An explicit type will help to follow manipulations better:
```scala
def factorial: Int => Int =
  (x: Int) => if (x == 0) 1 else x * factorial(x - 1)
```

And before we started to do anything, I want to pay attention to the perception of the recursion itself.
Good CS curriculums bring lambda calculus and explain recursion as self-reference that could be expressed by a few manipulations. Those manipulations are required because lambda calculus has a very simple and limited ruleset; self-reference is not part of that limited set. Those who studied recursions in terms of some modern programming language have a chance to perceive any recursion function as something from [self-reference paradoxes list](https://en.wikipedia.org/wiki/List_of_paradoxes#Self%E2%80%93reference). From such a perspective, recursion works like a miracle and it is better to not think a lot about it.
The common target of the next steps is to gain a better understanding of the internal structure and stop interpreting recursion as a black box (inside blackbox inside blackbox inside ...).

## saying goodbye to self-reference
Well, let's return to `factorial` and try to withdraw self-reference.
Why did we suppose to do that? The experiment there is doing the simplest possible step to not call `factorial` from `factorial`. We probably will need to do connections in some different ways after.

I loved the semi-joke by [Michael Vanier](https://mvanier.livejournal.com/2897.html):
>It's a tried-and-true principle of functional programming that if you don't know exactly what you want to put somewhere in a piece of code, just abstract it out and make it a parameter of a function.

<details markdown="block">
<summary markdown="span">More precise explanation via lambda calculus</summary>

[Recursive Lambda Functions the Y-Combinator](https://sookocheff.com/post/fp/recursive-lambda-functions/)
Let’s use the idea of a fixed-point function to help solve our addition problem using recursion. We already know how to use a function in lambda calculus: *function application*. Application involves substituting a function’s bound variables (arguments) with argument expressions and evaluating the function’s body. You can delay this application by wrapping your function in another function. For example,  the function
```
f : a
```  
is equivalent to the following function
```
λg.(g:a):f 
```
where the original function *f* becomes the argument of a new function application in the body of *g* — when *g* is applied, the return value is *f a*. By using *g* in place of f* in a recursive function, we can substitute the recursive call with a new function that does not recurse.

</details>


Let's replace `factorial` call in the body with `kernel` (it is not a full-featured factorial yet).
```scala
(x: Int) => if (x == 0) 1 else x * kernel(x - 1)
```

And `kernel` is our parameter now, it takes a number, and returns a number too.
```scala
def factorialWithKernel(kernel: Int => Int): Int => Int =
  (x: Int) => if (x == 0) 1 else x * kernel(x - 1)
```
Well, we produced *something*. `factorialWithKernel` is not recursive.
Some part of "magic" had been withdrawn though.
We decoupled self-reference and now the kernel is not calling kernel. At least, `factorialWithKernel` *simply calls* `kernel`. We don't know what kernel actually does.
I suggest playing around with `factorialWithKernel` before we start to return magic back.
We need to understand what we really want from our "simple" `kernel` function.

<details>
<summary>Let's add more logging</summary>
We need more evidence!
```scala
def factorialWithKernel(kernel: Int => Int): Int => Int =
  n => {
    if (n == 0) {
      println("returning 1!")
      1
    } else {
      val kernelResult = kernel(n - 1)
      val finalResult = n * kernelResult
      println(s"n[$n] * kernel($n - 1)[$kernelResult] = $finalResult")
      finalResult
    }
  }
```
</details>

## what is kernel?
What we can use as the simplest function? The identity function is our friend here.

```scala
val `factorialWithKernel(identity)` = factorialWithKernel(identity)
```

So, what we will have for zero?
```scala
`factorialWithKernel(identity)`(0)
```
Predictalby, `(x == 0)` condition serves our need and returns 1.

Should anybody care about kernel then?

```scala
val `factorialWithKernel(42)`: Int => Int = 
  factorialWithKernel(_ => 42)  
  
val result0With42Kernel = `factorialWithKernel(42)` apply 0  
println(s"Result for 0 applied to factorialWithKernel(42): $result0With42Kernel") 
```

`factorialWithKernel` takes care of the 0 input case and ignores kernel:
```
returning 1!
Result for 0 applied to factorialWithKernel(42): 1
```

What about passing non-zero?
```scala
val result1With42Kernel = 
  `factorialWithKernel(42)` apply 1  
  
println(s"Result for 1 applied to factorialWithKernel(42): $result0With42Kernel")
```

There is no "returning 1!" message and `kernel(0)` returned 42:
```
n[1] * kernel(1 - 1)[42] = 42
Result for 1 applied to factorialWithKernel(42): 42
```

So, calculations are broken. The `kernel` returns 42 and that's quite expected.
Can we tackle that?
What do we want in the **1**-as-input case?
Unconsciously it seems like we already put most of necessary lines.
1. We want to decrement **1**. Done? `n - 1` is there! Done.
2. We pass decremented value to the `kernel` of type `Int => Int`. We did it, **0** is passed!
3. We want `kernel` to calculate 1 for **0**-input but our kernel is `f(x) = 42`.

Apparently, we have a problem with kernel. Do we have function that returns **1** for **0**-input (`f(0) == 1`)?

It is scary to say, but `factorialWithKernel(42)` does it.
`factorialWithKernel` helps to utilize any `Int -> Int` function, nothing prevents us to give it a try.

```scala
val factorialWithDerivedKernel =
  factorialWithKernel(`factorialWithKernel(42)`)
```

`factorialWithDerivedKernel apply 1` works good!
```
returning 1!
n[1] * kernel(1 - 1)[1] = 1
```

Are we good? No. We need to generate more kernels for different inputs, we still reach 42 at the second iteration.

```
Let's apply 11 to factorialWithDerivedKernel
n[10] * kernel(10 - 1)[42] = 420
n[11] * kernel(11 - 1)[420] = 4620
Result for 11 applied to factorialWithDerivedKernel: 4620
```

## understanding properties of kernel
Now we understand what we missed once moved to non-recursive `factorialWithKernel` with `kernel` parameter.
`kernel` itself is missed!
We somehow don't care about what it particularly does. But still, it should be generated and passed and we barely can do it on our own.

Previously, we already noticed that `kernel` is something that pitentially could be derived. It hides self-referential *structure* instead of any particular piece of logic.
In practice, we see that result of factorialWithKernel is `Int => Int` function that does quite the same as `kernel` itself. We can (and should) *use those produced functions as kernels*.
It sounds recursive too, but it is not a big surprise that we ended up with a demand to turn `factorialWithKernel` into the function that behaves like kernel (but with allowance to refer to itself).


1. We have factorialWithKernel that takes `kernel`. The kernel should be a full-featured factorial function under the hood; reminding us that we use it to calculate the factorial of a decremented number: `kernel(x - 1)`.
2. It seems like we need some kind of tool that will be able to cook that function for us. On one side, we attempted to pass some dummy values and it almost worked. On another side, even with more manual manipulations, it is not clear how to assemble that recursion-without-recursion function.

Let's return to point number one.
We're passing the `kernel`. And we expect as a result is the same function that takes numbers and calculates factorial.

And that is the most mind-blowing thing.
We need such `kernel` that we can pass to `factorialWithKernel`  and receive back the same `kernel` function. Again, `factorialWithKernel(kernel)` should be equal to `kernel` itself.

<details markdown="block">
<summary markdown="span">Equality of functions</summary>

[Equality of Functions](https://mathstats.uncg.edu/sites/pauli/112/HTML/secfuneq.html) rule.
Two functions are equal if they have the same domain and codomain and their values are the same for all elements of the domain.
</details>

An attempt to manually derive the kernel works badly here. It is hard to think about equality once the second function is derived from the first one and that second function can calculate factorial for **1** while the first returns a dummy value.
When you do not write the functions down and just explore properties then the `kernel` is just a function that calculates the factorial of a decremented number. And the same function is returned.

Now, it time to recall fix-points.

>**fixed point** (also fixpoint or invariant point) of a function is an element that is mapped to itself by the function.
>**fixed-point combinator** is a higher-order function that returns some fixed point of its argument function.

At that particular moment, things are joining together. We need fixed-point combinator to provide `kernel`!

It doesn't clear how it could work. But we can try to move step by step with the hope that iterative updates will work in the end.

## Y Combinator

### Initial steps
Good. Step number one. Set up the signature.
We want to pass our `factorialWithKernel` and receive simple `Int => Int` back, let's just put the appropriate signature.
```scala
def fix[A](f: (A => A) => (A => A)): A => A = ???
```
I am replacing `Int` by generic since it just makes lines shorter.

Now we can start building definition.
We need to calculate *something*, easy step.
```scala
def fix[A](f: (A => A) => (A => A)): A => A = {  
  val calculated = ???
  calculated  
}
```

One more naive step. We should invoke `f`, aren't we?
```scala
def fix[A](f: (A => A) => (A => A)): A => A = {  
  val calculated = f(???) 
  calculated  
}
```

And what we can provide? Our attempts to provide some particular value failed previously. At that point, we need to introduce indirect self-reference, which had been withdrawn once we replaced self-reference with the `kernel`.
```scala
def fix[A](f: (A => A) => (A => A)): A => A = {  
  val calculated = f(calculated) 
  calculated  
}
```

Does it work? No.
The compiler says: "Recursive value calculated needs type". Not a big deal.
```scala
def fix[A](f: (A => A) => (A => A)): A => A = {  
  val calculated: A => A = f(calculated) 
  calculated  
}
```

What now? "Forward reference to value calculated extends over definition of value calculated". `calculated` should be calculated first to be passed as an argument for `f` function that calculates calculated.

We continue to leverage Scala's strengths. Laziness could help there.
```scala
def fix[A](f: (A => A) => (A => A)): A => A = {  
  lazy val calculated: A => A = f(calculated)
  calculated  
}
```

It compiles!

### Fighting with overflows

```scala
val factorial: Int => Int = fix(factorialWithKernel)
```
Without even running `factorial` we meet `java.lang.StackOverflowError`.

The problem here is `f`. We need to pass an argument to it. That argument needs the result of the evaluation. `f: (A => A) => (A => A)` has function as argument. And that argument is passed by value. That means we need to calculate our `kernel` before passing it into `factorialWithKernel`. It worked well when we passed our `_ => 42` and manually derived functions.
But it doesn't work when the orchestrator calls `f(calculated)` and `calculated` should be calculated *before* calling `f`.
`f(f(f(f(...))))` can't reach a point of return to finally pass any argument.
Again, we cat try to invoke lazy evaluation.
We need to amend the argument of `f` (which is of type `(A => A)`) and mark it as the passed-by-name parameter.

```scala
def fix[A](f: (=> (A => A)) => (A => A)) = {  
  lazy val calculated: A => A = f(calculated)
  calculated  
}
```

Anonymous definition without explicit typing will look as previously:
```scala
fix[Int](kernel => x => if (x == 0) 1 else x * kernel(x - 1))
```

Our functions going to look like this now:
```scala
def factorialWithKernelPassedByName(kernel: => (Int => Int)): Int => Int =  
  n => {  
    if (n == 0) {  
      println("returning 1!")  
      1  
    } else {  
      val kernelResult = kernel(n - 1)  
      val finalResult = n * kernelResult  
      println(s"n[$n] * kernel($n - 1)[$kernelResult] = $finalResult")  
      finalResult  
    }  
  }
```

Finally, it works as expected.
```scala
val fixedFactorial: Int => Int = 
  fix[Int](factorialWithKernelPassedByName)
```

```
Let's apply 11 to fixedFactorial
returning 1!
n[1] * kernel(1 - 1)[1] = 1
n[2] * kernel(2 - 1)[1] = 2
n[3] * kernel(3 - 1)[2] = 6
n[4] * kernel(4 - 1)[6] = 24
n[5] * kernel(5 - 1)[24] = 120
n[6] * kernel(6 - 1)[120] = 720
n[7] * kernel(7 - 1)[720] = 5040
n[8] * kernel(8 - 1)[5040] = 40320
n[9] * kernel(9 - 1)[40320] = 362880
n[10] * kernel(10 - 1)[362880] = 3628800
n[11] * kernel(11 - 1)[3628800] = 39916800
Result for 11 applied to fixedFactorial: 39916800
```

The `fix` manages to perform a pseudo-self-reference trick and "exit on time" without cycling.

### Built-in syntax
Explicit notes.
1. Scala provides native syntax to define a non-recursive function with self-reference
```scala
val factorial: Int => Int = new (Int => Int) { kernel => 
  def apply(n: Int): Int = 
    if (n == 1) 1 else n * kernel.apply(n - 1)  
}
```
So, you create an instance of `Int => Int`, and implement `apply` of `scala.Function1`.
`kernel` there is a self-reference we can use in the definition. You can check out [SO: Explicit self-references](https://stackoverflow.com/questions/8073263/explicit-self-references-with-no-type-difference-with-this) discussion in case such kind of self-reference require explicit explanation.

And still, in Scala, there is no explicit demand to avoid recursive definitions.
The initial function does the same thing and looks neat.
```scala
def factorial(x: Int): Int = 
  if (x == 0) 1 else x * factorial(x - 1)
```
Are we good with it then?
No! In both cases, we have the "opportunity" to overflow the stack. Y Combinator doesn't help with it.
In practice, we need to look for ways to write tail recursion.
It is easy to rewrite factorial to make it stack-safe.
```scala
def factorial(x: Int): BigInt = {  
  @tailrec  
  def factorialIteration(current: Int, accumulator: BigInt): BigInt =  
    if (current == 0) accumulator  
    else factorialIteration(current - 1, current * accumulator)  
  
  factorialIteration(x, accumulator = 1)  
}
```


## Tail recursions and monads
### tailRecM and explicit state for each iteration
In real applications, we typically need to handle some IO. `cats.FlatMap.tailRecM` is our friend there.

`def tailRecM[A, B](a: A)(f: A => F[Either[A, B]]): F[B]` takes initial argument and function that either calculates accumulator `A` state or returns `B` that ends recursion.

Let us have a state with the current number and accumulator. We need it to not play around with tuples and those `_1` and `_2` properties.
```scala
case class FactorialStepState(current: Int, accumulator: BigInt)
```

And we need an initial state.
```scala
object FactorialStepState {  
  val initial: Int => FactorialStepState = 
    x => FactorialStepState(x, 1)  
}
```

Now factorial looks like this.
```scala
def factorial(x: Int): BigInt =  
  FlatMap[Id].tailRecM[FactorialStepState, BigInt(FactorialStepState.initial(x))(state =>  
  if (state.current == 0) state.accumulator.asRight  
  else  
    FactorialStepState(  
      current = state.current - 1,  
      accumulator = state.current * state.accumulator  
    ).asLeft  
)
```
And once state itself is connected to some particular approach, it is ok to add specific functions to the state class iteself.
```scala
case class FactorialStepState(current: Int, accumulator: BigInt) {  
  
  /** Generates state for next iteration (in terms of algorithm that decreases a value on each step)  
    * @return updated state  
    */  
  def multiplyAndDecrement: FactorialStepState =  
    this.copy(current = this.current - 1, accumulator = this.current * this.accumulator)  
}
```

Now it is simplified.
```scala
def factorial(x: Int): BigInt =  
  FlatMap[Id].tailRecM[FactorialStepState, BigInt](FactorialStepState.initial(x))(state =>  
    if (state.current == 0) state.accumulator.asRight 
    else state.multiplyAndDecrement.asLeft  
  )
```

Often, we can gain more transparency after adding explicit state descriptions.
Attempts to give meaningful names and write accurate functions nudge to be very specific.
In factorial case, `current` and `accumulator` are just generic names for any kind of folding iteration. So, that's like using `i` in while/for loop. It would be better to use anything meaningful. And that's not easy in our case. [There are only two hard things in Computer Science: cache invalidation and naming things.](https://martinfowler.com/bliki/TwoHardThings.html)
We have some accumulator, it aggregates results of multiplication but there is no specific name for such kind of multiplication.
We have "factorial" itself but it is the multiplication of numbers in ascending order. But once we start to care more about transparency, there are no obstacles to amending the algorithm.
We can start from 1 and iteratively multiply numbers in descending orders to have meaningful values.
`number: Int` and `factorial: BigInt` have much more sense, also we can return all calculated factorials together as `factorials: Map[Int, BigInt]`.
Naturally, very mechanical `multiplyAndDecrement` could be replaced by a high-level `nextFactorial` function.
Now we have more maintainable and self-descriptive structure
```scala
case class Factorials(
  number: Int, 
  factorial: BigInt, 
  factorials: Map[Int, BigInt]
) {  
  
  /** Generates state for next iteration (in terms of algorithm that increases a value on each step)  
    * @return  
    *   updated state  
    */  
  def nextFactorial: Factorials = {  
    val incremented: Int   = this.number + 1  
    val calculated: BigInt = this.factorial * incremented  
    println(s"Factorial for $incremented is $calculated")  
    this.copy(  
      number = incremented,  
      factorial = calculated,  
      this.factorials + (incremented -> calculated)  
    )  
  }  
}  
  
object Factorials {  
  val initial: Factorials = Factorials(0, 1, Map(0 -> 1))  
}  
  
def factorialsUntil(x: Int): Map[Int, BigInt] =  
  FlatMap[Id].tailRecM[Factorials, Map[Int, BigInt]](Factorials.initial)(state =>  
    if (state.number == x) state.factorials.asRight else state.nextFactorial.asLeft  
  )
```