# Chapter 5 - Monad Transformers
[Back to index](index.md)

## Monad transformers in Cats
By convention, Cats provides a monad transformer `FooT` for a monad `Foo`. Eg:
`OptionT` is a transformer for the `Option` monad, `EitherT` for `Either`, 
`WriterT` for `Writer` and so on.

The transformer itself represents the inner monad in a stack, while the first
type parameter specifies the outer monad.

### Aside: Kleisly arrow
`ReaderT` is in fact a type alias for `Kleisli`.

## Type parameters
Many monads and all transformers have at least two type parameters, so we often
have to define type aliases for intermediate stages. Alternatively, we can use
the *kind projector* SBT plugin.

### Type aliases
Suppose we want to wrap `Either` around `Option`. `Either` has two type 
parameters while monads only have one. Hence, we need a type alias to convert 
the type constructor to the correct shape:

```scala
import cats.data.OptionT

type ErrorOr[A] = Either[String, A]

// wraps an Either[String, Option[A]] value
type ErrorOrOption[A] = OptionT[ErrorOr, A]
```

Similarly, if we want to create a `Future` of an `Either` of `Option`:

```scala
import cats.data.{EitherT, OptionT}
import scala.concurrent.Future

// wraps a Future[String Either A] value
type FutureEither[A] = EitherT[Future, String, A]

// wraps a Future[String Either Option[A]] value
type FutureEitherOption[A] = OptionT[FutureEither, A]
```

### Kind projector plugin
An alternative to using type aliases is to use the *kind projector* SBT plugin.
We do so by adding the following to our `build.sbt`:

```scala
addCompilerPlugin("org.spire-math" %% "kind-projector" % "0.9.4")
```

We can now wrap `Either` around `Option` as follows:

```scala
import cats.data.OptionT

// wraps an Either[String, Option[A]] value
type EitherOption[A] = OptionT[Either[String, ?], A]
```

And create a `Future` of an `Either` of `Option` with:

```scala
import cats.data.{EitherT, OptionT}
import scala.concurrent.Future

// wraps a Future[String Either Option[A]] value
type FutureEitherOption[A] = OptionT[EitherT[Future, String, ?], A]
```

## Default instances
Many monads in Cats are defined using the corresponding transformer and the 
`Id` monad. `Reader`, `Writer` and `State` are all defined this way. 

```scala
type Reader[E, A] = ReaderT[Id, E, A] // = Kleisli[Id, E, A]
type Writer[W, A] = WriterT[Id, W, A]
type State[S, A] = StateT[Id, S, A]
```
