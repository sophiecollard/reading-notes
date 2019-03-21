# Chapter 4 - Monads
[Back to index](index.md)

## Monads
A monad is a mechanism for sequencing computation. Monadic behaviour is captured by the following two operations:
  * `pure` of type `A => F[A]`
  * `flatMap` of type `(F[A], A => F[B]) => F[B]`

The `Monad` type class in Cats extends `FlatMap` (which provides the `flatMap` method) and `Applicative` (which provides the `pure` method). `Applicative` also extends `Functor`, giving every `Monad` a `map` method.

```scala
trait Functor[F[_]] {
  def map[A, B](fa: F[A])(f: A => B): F[B]
}

trait Applicative[F[_]] extends Functor[F] {
  def pure[A](value: A): F[A]
}

trait FlatMap[F[_]] {
  def flatMap[A, B](fa: F[A])(f: A => F[B]): F[B]
}

trait Monad[F[_]] extends Applicative[F] with FlatMap[F]
```

Note that the `map` method inherited from `Functor` can be implemented in terms of `flatMap` and `pure`:

```scala
def map[A, B](fa: F[A])(f: A => B): F[B] = {
  flatMap(fa)(x => pure(f(x)))
}
```

### Laws
Monads must comply with *left identity*, *right identity* and *associativity* laws. Concretely, the following must evaluate to `true`:

```scala
// Left identity: Calling pure and transforming the result with f is the same as calling f
pure(a).flatMap(f) == f(a)
// Right identity: Transforming with pure returns the monad unchanged
fa.flatMap(pure) == fa
// Associativity: Transforming with two functions f and g is the same as transforming with f
// and the transforming with g
fa.flatMap(f).flatMap(g) == fa.flatMap(x => f(x).flatMap(g))
fa.flatMap(f).flatMap(g) == fa.flatMap(x => for { y <- f(x); z <- g(y) } yield z)
```

## Id
Cats defines a type alias `Id` and provides type class instances such as `Functor[Id]` and `Monad[Id]` for it:

```scala
type Id[A] = A

object Id {
  def pure[A](value: A): Id[A] =
    value

  def map[A, B](fa: Id[A])(f: A => B): Id[B] =
    f(fa)

  def flatMap[A,B](fa: Id[A])(f: A => Id[B]): Id[B] =
    f(fa)
}
```

The ability to abstract over monadic code is extremely powerful. For example, we can run code asynchronously in production using `Future` and synchronously in test using `Id`.

## MonadError
Cats provides a `MonadError` type class that abstracts over `Either`-like data types:

```scala
trait MonadError[F[_], E] extends Monad[F] {
  // Lifts an error into the F context
  def raiseError[A](e: E): F[A]

  // Handle error, potentially recovering from it
  def handleError[A](fa: F[A])(f: E => A): F[A]

  // Test an instance of F, failing if the predicate is not satisfied
  def ensure[A](fa: F[A])(e: E)(f: A => Boolean): F[A]
}
```

Here's an example of how we instantiate this type class for `Either`:

```scala
import cats.MonadError
import cats.instances.either._

type ErrorOr[A] = Either[String, A]

val monadError = MonadError[ErrorOr, String]
```

The `raiseError` method is analogous to `pure` except that it's used to create an instance of an error:

```scala
val success = monadError.pure(42)
// success: ErrorOr[Int] = Right(42)

val failure = monadError.raiseError("Badness")
// failure: ErrorOr[Nothing] = Left("Badness")
```

## Eval
`Eval` abstracts over different models of evaluation:
  * eager (`Now` subtype, equivalent to `val` declarations)
  * lazy (`Always` subtype, equivalent to `def` declarations)
  * memoized (`Later` subtype, equivalent to `lazy val` declarations)

`Eval` has a `memoize` method that allows us to memoize a chain of computations. The result of the chain up to the call to memoize is cached, whereas calculations after the call retain their original semantics.

### Trampolining and Eval.defer
One useful property of `Eval` is that its `map` and `flatMap` methods are *trampolined*. This means we can nest calls to `map` and `flatMap` arbitrarily without consuming stack frames.

For instance, with factorial:

```scala
def factorial(n: BigInt): Eval[BigInt] = {
  if (n <= 1) {
    Eval.now { n }
  } else {
    Eval.defer { factorial(n - 1).map(_ * n) }
  }
}
```

Bear in mind that trampolining is not free. It avoids consuming stack frames by creating a chain of function objects on the heap. There are still limits on how deeply we can nest computations, but they are bounded by the size of the heap rather than the stack.

## Writer
`cats.data.Writer` is a monad that lets us carry a log along with a computation. We can use it to record messages, errors, or additional data about a computation and extract the log alongside the final result. A `Writer[W, A]` carries two values: a log of type `W` and a result of type `A`.

Note that Cats implements the `Writer` monad in terms of the monad transformer `WriterT`, so an instance of `Writer[W, A]` would actually appear as `WriterT[Id, W, A]` in the console.

```scala
type Writer[W, A] = WriterT[Id, W, A]
```

### Logging
A common use of writers is recording sequences of steps in multi-threaded computations where standard imperative logging techniques can result in interleaved messages from different contexts. With `Writer`, a computation's log is tied to the result such that we can run concurrent computations without mixing their logs.

We can wrap a result value `A` in a `Writer[W, A]` instance using `pure`. We must have a `Monoid[W]` instance in scope to do this so that Cats knows how to produce an empty log:

```scala
import cats.data.Writer
import cats.instances.vector._ // for Monoid[Vector]
import cats.syntax.applicative._ // for pure

type Logged[A] = Writer[Vector[String], A]

123.pure[Logged]
// res0: Logged[Int] = WriterT((Vector(), 123))
```

Similarly, we can wrap a log `W` in a `Writer[W, Unit]` with the `tell` method:

```scala
import cats.data.Writer
import cats.syntax.writer._ // for tell

Vector("msg1", "msg2", "msg3").tell
// res0: WriterT[Id, Vector[String], Unit] = WriterT((Vector(msg1, msg2, msg3), ()))
```

We can extract the result `A` and log `W` from a `Writer[W, A]` instance using the `value` and `written` methods, respectively. We can extract both values as a tuple using `run`.

```scala
final case class Writer[W, A](run: Tuple2[W, A]) {
  val log: W = run._1
  val value: A = run._2
}
```

### Composition
The log in a `Writer` is preserved when we `map` or `flatMap` over it. `flatMap` *appends* the logs from the source `Writer` and the result of the user’s sequencing function. For this reason it’s good practice to use a log type that has an efficient append and concatenate operations, such as a `Vector`.

```scala
import cats.instances.vector._ // for Monoid[Vector]
import cats.syntax.applicative._ // for pure
import cats.syntax.writer._ // for tell and writer

val writer = for {
  a <- 10.pure[Logged]
  _ <- Vector("msg1").tell
  b <- 32.writer(Vector("msg2"))
} yield a + b
// res0: WriterT[Id, Vector[String], Int] = WriterT((Vector(msg1, msg2), 42))
```

## Reader
`cats.data.Reader` is a monad that allows us to sequence operations that depend on some input.

A `Reader[A, B]` is constructed from a function `A => B` using the `Reader.apply` constructor.

```scala
final case class Reader[A, B](run: A => B)
```

### Dependency injection
A common use of readers is dependency injection. If we have a number of operations that depend on some input, we can chain them together using a reader to produce one large operation that accepts the input as a parameter and runs our program in the order specified.

There are many ways of implementing dependency injection in Scala, from simple techniques like methods with multiple parameter lists, through implicit parameters and type classes, to complex techniques like the cake pattern and DI frameworks.

### Composition
The `flatMap` reader method lets us combine readers that depend on the same input type:

```scala
import cats.data.Reader

final case class Cat(name: String, favouriteFood: String)
val cat = Cat("Garfield", "lasagne")

val greetKitty: Reader[Cat, String] = Reader(cat => s"Hello ${cat.name}.")
val feedKitty: Reader[Cat, String] = Reader(cat => s"Have a nice bowl of ${cat.favouriteFood}.")

val greetAndFeed: Reader[Cat, String] = {
  for {
    greetOutput <- greetKitty
    feedOutput <- feedKitty
  } yield s"$greetOutput $feedOutput"
}

greetAndFeed(cat)
// res0: cats.Id[String] = Hello Garfield. Have a nice bowl of lasagne.
```

## State
`cats.data.State` allows us to pass additional state around as part of a computation. It allows us to model mutable state in a purely functional way, without using mutation. An instance of `State` is a function that does two things:
  * transforms an input state of type `S` to an output state of the same type
  * computes a result of type `A`

We can "run" a state monad by supplying an initial state. In order to ensure stack safety, the result is wrapped in an `Eval`.

```scala
import cats.Eval

final case class State[S, A](f: S => (S, A)) {
  def run(state: S): Eval[(S, A)] = 
    Eval.now(f(state))

  def runA(state: S): Eval[A] =
    Eval.now(f(state)._2)

  def runS(state: S): Eval[S] =
    Eval.now(f(state)._1)
}
```

### Composition
Once again, the `flatMap` method lets us combine state instances with the same type of state and result. In the example below, the state is threaded from step1 to step2 even though we don’t interact with it in the for comprehension.

```scala
import cats.data.State

val step1 = State[Int, String] { n =>
  val m = n + 1
  (m, s"The result of step 1 is: $m")
}

val step2 = State[Int, String] { n =>
  val m = n * 2
  (m, s"The result of step 2 is: $m")
}

val allSteps = for {
  a <- step1
  b <- step2
} yield (a, b)

val (state, result) = allSteps.run(20).value
// state: Int = 42
// result: (String, String) = (The result of step 1 is: 21, The result of step 2 is: 42)
```

### Post-order calculator example
An interpreter for a post-order calculator reduces expressions such as `1 2 + 3 *`. To do so, it traverses symbols from left to right, carrying a stack of operands as it goes. When it sees a number, it pushes it onto the stack. When it sees an operator, it pops two operands off the stack, operates on them, and pushes the result back onto the stack.

We start by implementing an `evalOne` function capable to evaluating a single symbol:

```scala
import cats.data.State

type CalcState[A] = State[List[Int], A]

/**
  * Updates the state by pushing a new integer n onto the stack.
  * The intermediate result returned is simply n.
  */
def operand(n: Int): CalcState[Int] = 
  State[List[Int], Int] { stack =>
    (n :: stack, n)
  }

/**
  * Computes a result by popping two integers from the stack and applying an operator to them.
  * Updates the state by pushing the result onto the stack.
  */
def operator(f: (Int, Int) => Int): CalcState[Int] =
  State[List[Int], Int] {
    case b :: a :: tail =>
      val result = f(a, b)
      (result :: tail, result)
    case _ =>
      sys.error("A failure occurred!")
  }

/**
  * Evaluates a single symbol, updating the state and result.
  */
def evalOne(input: String): CalcState[Int] =
  input match {
    case "+" => operator(_ + _)
    case "-" => operator(_ - _)
    case "*" => operator(_ * _)
    case "/" => operator(_ / _)
    case n   => operand(n.toInt)
  }
```

Below are a couple of examples of `evalOne` invocation.

```scala
val (state, result) = evalOne("42").run(Nil).value
// state: List[Int] = List(42)
// result: Int = 42

val program: CalcState[Int] =
  for {
    _ <- evalOne("1")
    _ <- evalOne("2")
    result <- evalOne("+")
  } yield result

val (state, result) = program.run(Nil).value
// state: List[Int] = List(3)
// result: Int = 3
```

We then implement an `evalAll` function to interpret the sequence of symbols in an expression:

```scala
/**
  * Evaluates all symbols in an expression.
  */
def evalAll(input: List[String]): CalcState[Int] =
  input match {
    case Nil =>
      // impossible case
      sys.error("Empty expression")
    case s :: Nil =>
      // base case
      evalOne(s)
    case s :: tail =>
      // recursive case
      evalOne(s).flatMap { _ =>
        evalAll(tail)
      }
  }
```

For simplicity, the `evalAll` method above can be re-written as:

```scala
import cats.syntax.applicative._ // for pure

def evalAll(input: List[String]): CalcState[Int] =
  input.foldLeft(0.pure[CalcState]) { (acc, nextSymbol) =>
    acc.flatMap(_ => evalOne(nextSymbol))
  }
```

Note that the initial value could have been `20.pure[CalcState]` or any other interger value lifted in the `CalcState` context. Unless the input is an empty list, this intermediary input will be discarded.

We can verify that the interpreter works as expected:

```scala
val (state, result) = evalAll(List("1", "2", "+", "3", "*")).run(Nil).value
// state: List[Int] = List(9)
// result: Int = 9
```
