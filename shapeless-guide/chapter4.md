# Chapter 4 - Working with Types and Implicits
[Back to index](index.md)

## Dependent types
A simplified definition of Shapeless' `Generic` type class is:

```scala
trait Generic[A] {
  type Repr
  def to(value: A): Repr
  def from(value: Repr): A
}
```

Now, consider the return type of the following `getRepr` method:

```scala
import shapeless.Generic

def getRepr[A](value: A)(implicit gen: Generic[A]) =
  gen.to(value)
```

That return type would depend on the `Generic` instance passed implicitly. Specifically, it would be the `Repr` type defined on that instance. This is known as _dependent typing_.

Note that, using `Aux`, we could re-write the signature of `getRepr` to have an explicit return type `R`. However, this would effectively render `getRepr` useless since we'd have to specify the `R` type parameter.

```scala
import shapeless.Generic

def getRepr[A, R](value: A)(implicit gen: Generic.Aux[A, R]): R =
  gen.to(value)
```

Type parameters (`A`) are useful _inputs_ and type members (`Repr`) useful _outputs_.

## Dependently typed functions
Shapeless uses dependent types all over the place. One example is the `Last` type class which returns the last element in an `HList`:

```scala
trait Last[L <: HList] {
  type Out
  def apply(in: L): Out
}
```

Consider implementing our own type class, `Second`, which returns the second element of an `HList`:

```scala
import shapeless.HList

trait Second[L <: HList] {
  type Out
  def apply(value: L): Out
}

object Second {
  type Aux[L <: HList, O] = Second[L] { type Out = O }

  def apply[L <: HList](implicit inst: Second[L]): Aux[L, inst.Out] =
    inst
}
```

Before we can use it, we must define a single instance of `Second` for `HList`s of at least two elements:

```scala
import Second._
import shapeless.{HList, ::}

implicit def hListSecond[A, B, R <: HList]: Aux[A :: B :: R, B] =
  new Second[A :: B :: R] {
    type Out = B
    def apply(value: A :: B :: R): B =
      value.tail.head
  }
```

We can then summon instances using `Second.apply` as follows:

```scala
import shapeless.{HNil, ::}

val second = Second[String :: Int :: Boolean :: HNil]
// res0: Second[String :: Int :: Boolean :: HNil] { type Out = Int } = ...

second("sundae" :: 1 :: false :: HNil)
// res1: second.Out = 1
```

`Second.apply` is a dependently typed function, i.e. it provides a means of calculating a type member (`Out`) from a type parameter (`L`).

### Aside: _summoner_ method vs _implicitly_ vs _the_
Note that the return type of `Second.apply` is `Aux[L, O]`, not `Second[L]`. This ensures that the apply method does not erease type members on summoned instances.

Compare the type of `Last` instances summoned via `implicitly`, `apply` and Shapeless' `the` method. Note how `implicitly` ereases the `Out` type member while `apply` and `the` don't. For this reason, we should avoid using `implicitly` when working with dependently typed functions.

```scala
import shapeless._
import shapeless.ops.hlist.Last

implicitly[Last[String :: Int :: HNil]]
// res0: Last[String :: Int :: HNil] = ...

Last[String :: Int :: HNil]
// res1: Last[String :: Int :: HNil]{type Out = Int} = ...

the[Last[String :: Int :: HNil]]
// res2: Last[String :: Int :: HNil]{type Out = Int} = ...
```

## Chaining dependent functions
Dependently typed functions provide a means of calculating one type from another. We could for instance use a `Generic` to calculate a `Repr` for a case class and use `Last` to get the type of the last field in that case class:

```scala
import shapeless._
import shapeless.ops.hlist.Last

def lastField[A, Repr <: HList](input: A)(
  implicit
  gen: Generic.Aux[A, Repr],
  last: Last[Repr]
): last.Out = last.apply(gen.to(input))
```

See additional example pages 48-49.

See guidelines page 50 to ensure dependently typed functions compile.
