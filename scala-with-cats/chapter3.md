# Chapter 3 - Functors
[Back to index](index.md)

## Functors
Informally, functors are defined as data containers that implement a `map` method. We can think of the `map` method as a way of sequencing computations on values inside this container, or context. Examples of functors include `List`, `Option` and `Future`.

Formally, a functor is a type `F[A]` with an operation `map` of type `(A => B) => F[B]`.

A simplified implementation of Cats' `Functor` type is:

```scala
import scala.language.higherKinds

trait Functor[F[_]] {
  def map[A,B](fa: F[A])(f: A => B): F[B]
}
```

### Laws
Functors must comply with *identity* and *compositional* laws. Concretely, the following must return true:

```scala
// Passing the identity function to map returns the functor contents unchanged
fa.map(identity) == fa
// Mapping with two functions f and g is the same as mapping with f and then mapping with g
fa.map(f).map(g) == fa.map(x => g(f(x)))
```

## Aside: Higher kinds and type constructors

### Higher kinds
*Kinds* are like types for types. They describe the number of type arguments in a type.

Whenever we declare a type constructor with `A[_]` syntax, we need to enable the higher kinded type language feature to suppress warnings from the compiler.

This can be done with an import:

```scala
import scala.language.higherKinds
```

Or by adding the following to `scalacOptions` in `build.sbt`:

```scala
scalacOptions += "-language:higherKinds"
```

### Type constructors
*Type constructors* take at least one *type parameter*. For instance, `List` is a *type constructor* that takes one type parameter while `List[A]` is a *type* produced using a type parameter `A`.

## Contravariant and invariant functors

### Contravariant functors
The *contravariant functor* provides a `contramap` operation that represents prepending an operation to a chain (as opposed to appending with a covariant functor). The type signature of `contramap` is:

```scala
trait F[A] {
  def contramap(f: B => A): F[B]
}
```

Cats defines the contravariant functor as:

```scala
trait Contravariant[F[_]] {
  def contramap[A,B](fa: F[A])(f: B => A): F[B]
}
```

The `contramap` method only makes sense for data types that represent *transformations*. The `format` method on a `Printable` type class is an example of such transformation:

```scala
trait Printable[A] { self =>
  def format(value: A): String

  def contramap[B](f: B => A): Printable[B] = {
    new Printable[B] {
      def format(value: B): String = {
        self.format(f(value))
      }
    }
  }
}
```

A new `Printable` instance for some new type `Box` can be defined using `contramap`:

```scala
final case class Box[A](value: A)

object Box {
  implicit def printable[A](implicit ev: Printable[A]): Printable[Box[A]] = {
    ev.contramap(_.value)
  }
}
```

### Invariant functors
Invariant functors implement an `imap` method. If `map` generates new type class instances by *appending* a function to a chain and `contramap` generates them by *prepending* an operation to a chain, `imap` generates them via a pair of *bidirectional transformations*. The type signature of `imap` is:

```scala
trait F[A] {
  def imap[B](f: A => B, g: B => A): F[B]
}
```

Cats defines the invariant functor as:

```scala
trait Invariant[F[_]] {
  def imap[A,B](fa: F[A])(f: A => B)(g: B => A): F[B]
}
```

Think of a `Codec` type class for encoding and decoding values:

```scala
trait Codec[A] { self =>
  def encode(value: A): String

  def decode(value: String): A

  def imap[B](enc: A => B, dec: B => A): Codec[B] = {
    new Codec[B] {
      def encode(value: B): String = self.encode(dec(value))

      def decode(value: String): B = enc(self.decode(value))
    }
  }
}
```

A new `Codec` instance can be defined for some type `Box` using `imap`:

```scala
final case class Box[A](value: A)

object Box {
  implicit def codec[A](implicit ev: Codec[A]): Codec[Box[A]] = {
    ev.imap(Box.apply, _.value)
  }
}
```

Consider a second example: We want to define a `Monoid[Symbol]` instance. We already have `Monoid[String]` and `Invariant[Monoid]` instances provided by Cats. We can do so with:

```scala
import cats.Monoid
import cats.instances.string._ // for Monoid[String]
import cats.syntax.invariant._ // for imap

implicit val symbolMonoid: Monoid[Symbol] = Monoid[String].imap(Symbol.apply)(_.name)
```

### Covariance, contravariance and invariance summary
If `F` is a *covariant* functor (eg: `Option`), whenever we have an `F[A]` and a function `A => B` we can always obtain an `F[B]`.

If `F` is a *contravariant* functor (eg: `Printable`), whenever we have an `F[A]` and a function `B => A` we can always obtain an `F[B]`.

Finally, invariant functors (eg: `Codec`) capture cases where we can convert between `F[A]` and `F[B]` via `A => B` and `B => A` functions.

## Aside: Partial unification
See pages 70-74.

## Summary
*Covariant functors* represent the ability to apply functions to a value in some context. Successive calls to `map` apply these functions in sequence, each accepting the result of its
predecessor as a parameter.

*Contravariant functors* represent the ability to prepend functions to a function-like context. Successive calls to `contramap` sequence these functions in the opposite order to `map`.

*Invariant functors* represent bidirectional transformations.
