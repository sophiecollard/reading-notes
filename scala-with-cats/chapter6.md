# Chapter 6 - Semigroupal and Applicative
[Back to index](index.md)

Not all program flows can be represented using functors and monads. One such example is form validation. `map` and `flatMap` aren’t quite suitable because they make the assumption that each computation is *dependent* on the previous one. However, when we validate a form we want to return *all* the errors to the user, not stop on the first one we encounter.

In this chapter, we look at two type classes that support program flows where computations are *independent* of one another: `Semigroupal` and `Applicative`.

## Semigroupal
While `Semigroup` allows us to join values, `Semigroupal` allows us to join contexts. A simplified implementation is of Cats' `Semigroupal` type class is:

```scala
trait Semigroupal[F[_]] {
  def product[A, B](fa: F[A], fb: F[B]): F[(A, B)]
}
```

Here are some examples using `Semigroupal` on `Option`:

```scala
import cats.Semigroupal
import cats.instances.option._

Semigroupal[Option].product(Some(123), Some("abc"))
// res0: Option[(Int, String)] = Some((123, abc))

Semigroupal[Option].product(Some(123), None)
// res1: Option[(Int, Nothing)] = None

Semigroupal.tuple3(Option(123), Option("abc"), Option(false))
// res2: Option[(Int, String, Boolean)] = Some(123, abc, false)

Semigroupal.map3(Option(1), Option(2), Option(3))(_ + _ + _)
// res3: Option[Int] = Some(6)

Semigroupal.map2(Option(1), Option.empty[Int])(_ + _)
// res4: Option[Int] = None
```

### Behaviour
`Semigroupal`'s `product` doesn’t always provide the behaviour we expect, particularly for types that also have instances of `Monad`. For `Future`, it provides *parallel* rather than *sequential* execution. For `List` however, it returns the cartesian product rather than a zipped list. And for `Either`, it provides the same *fail-fast* semantics as `flatMap`.

```scala
import cats.Semigroupal
import cats.instances.either._

Semigroupal[Either[Vector[String], ?]].product(
    Left(Vector("Error 1")),
    Left(Vector("Error 2"))
)
// res0: Either[Vector[String], (Nothing, Nothing)] = Left(Vector("Error 1"))
```

The reason for the surprising results for `List` and `Either` is that they are both monads. To ensure consistent semantics, Cats’ `Monad` (which extends `Semigroupal` via `Applicative`) provides a standard definition of `product` in terms of `map` and `flatMap`.

## Apply syntax
Cats provides an *apply* syntax that provides a shorthand for some of the examples in the previous section.

### Tupled
The `tupled` method uses a `Semigroupal` instance (`Semigroupal[Option]` in the example below) to zip values in a context into a tuple, similarly to `Semigroupal`'s `product` and `tuple*` methods.

```scala
import cats.instances.option._ // for Semigroup[Option]
import cats.syntax.apply._ // for tupled

(Option(123), Option("abc"), Option(false)).tupled
// res0: Option[(Int, String, Boolean)] = Some((123, abc, false))
```

### MapN
The `mapN` method uses `Semigroupal` and `Functor` instances to apply a function to values in a context, similarly to `Semigroupal`'s `map*` methods.

```scala
import cats.instances.option._ // for Semigroup[Option]
import cats.syntax.apply._ // for mapN

final case class Cat(name: String, yearOfBirth: Int, colour: String)

(
    Option("Garfield"),
    Option(1978),
    Option("orange")
).mapN(Cat.apply)
// res0: Option[Cat] = Some(Garfield, 1978, orange)
```

## Validated
We can't design a monadic data type that implements error accumulating semantics without breaking the consistency of `map` and `flatMap`, both of which assume that each computation is dependent on the previous one. Luckily, Cats provides a `Validated` data type that has an instance of `Semigroupal` but no instance of `Monad`. Its implementation of `product` is therefore free to accumulate errors.

```scala
import cats.Semigroupal
import cats.data.Validated
import cats.instances.vector._ // for Semigroup[Vector]

type AllErrorsOr[A] = Validated[Vector[String], A]

Semigroupal[AllErrorsOr].product(
    Validated.invalid(Vector("Error 1")),
    Validated.invalid(Vector("Error 2"))
)
// res0: AllErrorsOr[(Nothing, Nothing)] = Invalid(Vector(Error 1, Error 2))
```

`Validated` has two subtypes, `Valid` and `Invalid`, which loosely correspond to `Either`'s `Right` and `Left`, respectively.

By converting between `Either` and `Validated` using Cats' `toValidated` and `toEither` methods, we can use a mix of `Either` to combine computations in sequence using fail-fast semantics and `Validated` to combine them in parallel using accumulating semantics.

### Composition
`Validated` accumulates errors using a `Semigroup`, so we need one of those in scope to summon `Semigroupal`. Otherwise, we get an annoyingly unhelpful compilation error.

For standard library types, such as `Vector`, we must import the corresponding `Semigroup` instance with:

```scala
import cats.instances.vector._
```

For types from the Cats library, such as `NonEmptyVector`, the necessary `Semigroup` instance is defined in the type's companion object. Hence, no import is required.

Once all the implicits to summon `Semigroupal` are in scope, we can use the apply syntax or any other `Semigroupal` methods to accumulate errors:

```scala
import cats.data.NonEmptyVector
import cats.syntax.apply._ // for tupled
import cats.syntax.validated._ // for invalid

(
    NonEmptyVector.of("Error 1").invalid[Int],
    NonEmptyVector.of("Error 2").invalid[Int]
).tupled
// res0: cats.data.Validated[cats.data.NonEmptyVector[String], (Int, Int)] = Invalid(NonEmptyVector(Error 1, Error 2))
```

`Validated` does not have a `flatMap` operation since it isn't a monad. However, Cats provides an `andThen` method with an identical type signature.

```scala
import cats.syntax.validated._ // for valid

final case class User(name: String, age: Int)

"Max".valid.andThen { name =>
    32.valid.map { age =>
        User(name, age)
    }
}
// res0: cats.data.Validated[Nothing, User] = Valid(User(Max, 32))

import cats.instances.vector._ // for Semigroup[Vector]
import cats.syntax.apply._ // for mapN

(
    "Max".valid[Vector[String]],
    32.valid[Vector[String]]
).mapN(User.apply)
// res1: cats.data.Validated[Vector[String], User] = Valid(User(Max, 32))
```

#### Collections for error accumulation
When accumulating errors, we should prefer collections with a constant-time append operation. This is why we use `Vector` instead of `List`.

## Apply and Applicative
`Semigroupal` provide a subset of the functionality of a related and more widely-used type class: the *applicative functor*, or `Applicative`. `Semigroupal` and `Applicative` effectively provide alternative encodings of the same no?on of joining contexts.

A simplified definition of Cats' `Applicative` is:

```scala
trait Apply[F[_]] extends Semigroupal[F] with Functor[F] {
    def ap[A, B](ff: F[A => B])(fa: F[A]): F[B]

    def product[A, B](fa: F[A], fb: F[B]): F[(A, B)] = {
        ap(map(fa)(a => (b: B) => (a, b)))(fb)
    }
}

trait Applicative[F[_]] extends Apply[F] {
    def pure[A](value: A): F[A]
}
```

The `pure` method in `Applicative` is the same as the one in `Monad`.

Note the tight relationship between `product`, `ap`, and `map` that allows any one of them to be defined in terms of the other two.

### Hierarchy of Sequencing Type Classes
See pages 160 - 162.
