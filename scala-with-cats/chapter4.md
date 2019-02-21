# Chapter 4 - Monads
[Back to index](index.md)

## Monads
A monad is a mechanism for sequencing computation. Monadic behaviour is captured by the following two operations:
  * `pure` of type `A => F[A]`
  * `flatMap` of type `(F[A], A => F[B]) => F[B]`

A simplified implementation of Cats' `Monad` is:

```scala
trait Monad[F[_]] {
  def pure[A](value: A): F[A]

  def flatMap[A,B](fa: F[A])(f: A => F[B]): F[B]
}
```

Every monad is also a functor. In fact, `map` can be implemented in terms of `flatMap` and `pure`:

```scala
trait Monad[F[_]] {
  // ...

  def map[A,B](fa: F[A])(f: A => B): F[B] = {
    flatMap(fa)(x => pure(f(x)))
  }
}
```

### Laws
Monads must comply with *left identity*, *right identity* and *associativity* laws. Concretely, the following must return true:

```scala
// Left identity: Calling pure and transforming the result with f is the same as calling f
pure(a).flatMap(f) == f(a)
// Right identity: Transforming with pure is the same as doing nothing
fa.flatMap(pure) == fa
// Associativity: Transforming with two functions f and g is the same as transforming with f
// and the transforming with g
fa.flatMap(f).flatMap(g) == fa.flatMap(x => f(x).flatMap(g))
fa.flatMap(f).flatMap(g) == fa.flatMap(x => for { y <- f(x); z <- g(y) } yield z)
```

## Monad in Cats
The `Monad` type class in Cats extends `FlatMap` (which provides the `flatMap` method) and `Applicative` (which provides the `pure` method). `Applicative` also extends `Functor` which gives every `Monad` a `map` method.

```scala
trait Functor[F[_]] {
  def map[A,B](fa: F[A])(f: A => B): F[B]
}

trait Applicative[F[_]] extends Functor[F] {
  def pure[A](value: A): F[A]
}

trait FlatMap[F[_]] {
  def flatMap[A,B](fa: F[A])(f: A => F[B]): F[B]
}

trait Monad[F[_]] extends Applicative[F] with FlatMap[F]
```

## Id
Cats defines a type alias `Id` and provides type class instances such as `Functor[Id]` and `Monad[Id]` for it:

```scala
type Id[A] = A

object Id {
  def pure[A](value: A): Id[A] = value

  def map[A,B](fa: Id[A])(f: A => B): Id[B] = {
    f(fa)
  }

  def flatMap[A,B](fa: Id[A])(f: A => Id[B]): Id[B] = {
    f(fa)
  }
}
```

The ability to abstract over monadic and non-monadic code is extremely powerful. For example, we can run code asynchronously in production using `Future` and synchronously in test using `Id`.

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
`Eval` abstracts over different models of evaluation with subtypes: eager (`Now`, equivalent to `val` declarations), lazy (`Always`, equivalent to `def` declarations) and memoized (`Later`, equivalent to `lazy val` declarations).

`Eval` has a `memoize` method that allows us to memoize a chain of computations. The result of the chain up to the call to memoize is cached, whereas calculations after the call retain their original semantics.

### Trampolining and Eval.defer
One useful property of `Eval` is that its `map` and `flatMap` methods are trampolined. This means we can nest calls to `map` and `flatMap` arbitrarily without consuming stack frames.

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

Bear in mind that trampolining is not free. It avoids consuming stack by creating a chain of function objects on the heap. There are still limits on how deeply we can nest computations, but they are bounded by the size of the heap rather than the stack.

## Writer
`cats.data.Writer` is a monad that lets us carry a log along with a computation. We can use it to record messages, errors, or additional data about a computation, and extract the log alongside the final result. A `Writer[W, A]` carries two values: a log of type `W` and a result of type `A`.

One common use for writers is recording sequences of steps in multi-threaded computations where standard imperative logging techniques can result in interleaved messages from different contexts. With `Writer` the log for the computation is tied to the result, so we can run concurrent computations without mixing logs.

Note that Cats implements the `Writer` monad in terms of the monad transformer `WriterT`, so an instance of `Writer[W, A]` would actually appear as `WriterT[Id, W, A]` in the console.

```scala
type Writer[W, A] = WriterT[Id, W, A]
```

### Pure, tell, value, written and run methods
We can wrap a result value `A` in a `Writer[W, A]` instance using `pure`. We must have a `Monoid[W]` instance in scope to do this so that Cats knows how to produce an empty log:

```scala
import cats.data.Writer
import cats.instances.vector._ // for Monoid[Vector]
import cats.syntax.applicative._ // for pure

type Logged[A] = Writer[Vector[String], A]

123.pure[Logged]
// res0: Logged[Int] = WriterT(Vector(), 123)
```

Similarily, we can wrap a log `W` in a `Writer[W, Unit]` with the `tell` method:

```scala
import cats.syntax.writer._ // for tell

Vector("msg1", "msg2", "msg3").tell
// res0: Writer[Vector[String], Unit] = WriterT(Vector("msg1", "msg2", "msg3"), ())
```

We can extract the result `A` and log `W` from a `Writer[W, A]` using `value` and `written` methods, respectively. We can extract both values as a tuple using `run`.

```scala
final case class Writer[W, A](run: Tuple2[W, A]) {
  val log: W = run._1
  val value: A = run._2
}
```

### Composition
The log in a `Writer` is preserved when we `map` or `flatMap` over it. `flatMap` appends the logs from the source `Writer` and the result of the user’s sequencing function. For this reason it’s good practice to use a log type that has an efficient append and concatenate operations, such as a `Vector`.

```scala
val writer = for {
  a <- 10.pure[Logged]
  _ <- Vector("msg1").tell
  b <- 32.writer(Vector("msg2"))
} yield a + b
// res0: Writer[Vector[String], Int] = WriterT(Vector("msg1", "msg2"), 42)
```

## Reader
`cats.data.Reader` is a monad that allows us to sequence operations that depend on some input.

A `Reader[A, B]` is constructed from a function `A => B` using the `Reader.apply` constructor.

```scala
final case class Reader[A, B](run: A => B)
```

### Dependency injection
One common use for readers is dependency injection. If we have a number of operations that all depend on some external configuration, we can chain them together using a Reader to produce one large operation that accepts the configuration as a parameter and runs our program in the order specified.

There are many ways of implementing dependency injection in Scala, from simple techniques like methods with multiple parameter lists, through implicit parameters and type classes, to complex techniques like the cake pattern and DI frameworks.

### Composition
The `flatMap` reader method lets us combine readers that depend on the same input type:

```scala
final case class Cat(name: String, favouriteFood: String)
val cat = Cat("Garfield", "lasagne")

val greetKitty: Reader[Cat, String] = Reader(cat => s"Hello ${cat.name}.")
val feedKitty: Reader[Cat, String] = Reader(cat => s"Have a nice bowl of ${cat.favouriteFood}.")

val greetAndFeed: Reader[Cat, String] = {
  for {
    greet <- greetKitty
    feed <- feedKitty
  } yield s"$greet $feed"
}

greetAndFeed(cat)
// res0: cats.Id[String] = Hello Garfield. Have a nice bowl of lasagne.

```

## State
`cats.data.State` allows us to pass additional state around as part of a computation. It allows us to model mutable state in a purely functional way, without using mutation. An instance of `State` is a function that does two things:
  * transforms an input state of type `S` to an output state of the same type
  * computes a result of type `A`

```scala
type State[S, A] = S => (S, A)
```

### Composition
Once again, the `flatMap` method lets us combine state instances with the same type of state and result.

See interpreter example pages 123 - 126.
