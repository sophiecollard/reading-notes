# CHAPTER 2: ADTs and generic representations
[back to index](index.md)

## Algebraic data types
Algebraic Data Types (ADTs) let us represent data using _and_ and _or_. For instance, a shape can be a rectangle or a circle. A rectangle has a width and a height. A circle has a radius.

```scala
sealed trait Shape
final case class Rectangle(width: Double, height: Double) extends Shape
final case class Circle(radius: Double) extends Shape
```

In ADT terminology, _and_ types (`Rectangle`, `Circle`) are referred to as _products_ and _or_ types (`Shape`) as _coproducts_.

Sealed traits and case classes are undoubtedly the most convenient encoding of ADTs in Scala. However, they aren’t the only encoding. We could have chosen `Tuple` and `Either` to encode products and coproducts, respectively:

```scala
type Rectangle = Tuple2[Double, Double]
type Circle = Double
type Shape = Either[Rectangle, Circle]
```

While the first example is more readable, the second is more generic. Any code that operates on a pair of doubles will be able to operate on the second definition of our `Rectangle` type, and vice-versa.

Instead of `Tuple` and `Either`, Shapeless uses its own generic data types to represent generic products and coproducts.

## Generic product encodings
Shapeless uses a generic encoding for products called a heterogeneous list. An `HList` is either the empty list `HNil`, or a pair `::[H, T]` where `H` is an arbitrary type and `T` is another `HList`.

The compiler knows the exact length of each `HList`, so it becomes a compilation error to take the head or tail of an empty list.

## Generic coproduct encodings
In Shapeless coproducts take the form `A :+: B :+: C :+: CNil` meaning _“A or B or C”_, where `:+:` can be loosely interpreted as `Either`. `:+:` has two subtypes, `Inl` and `Inr`, that correspond loosely to `Left` and `Right`. We can’t instantiate `CNil` or build a coproduct purely from instances of `Inr`. We always have an `Inl` in a value.

```scala
import shapeless.{Coproduct, CNil, Inl, Irn, :+:}

case class Red()
case class Amber()
case class Green()

type Light = Red :+: Amber :+: Green :+: CNil

val red: Light = Inl(Red(())
val green: Light = Inl(Inr(Inl(Green())))
```

## Generic type class
Shapeless provides a type class called `Generic` that allows us to switch back and forth between a concrete ADT and its generic representation.

```scala
import shapeless.Generic

case class IceCream(
  name: String,
  numCherries: Int,
  inCone: Boolean
)

val iceCreamGen = Generic[IceCream]

val iceCream = IceCream("Sundae", 1, false)
val repr = iceCreamGen.to(iceCream)
// repr: iceCreamGen.Repr = Sundae :: 1 :: false :: HNil
val iceCream2 = iceCreamGen.from(repr)
// iceCream2: IceCream = IceCream(Sundae, 1, false)
```
