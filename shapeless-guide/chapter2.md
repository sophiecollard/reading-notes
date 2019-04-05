# Chapter 2 - ADTs and Generic Representations
[Back to index](index.md)

## Algebraic data types
Algebraic Data Types (ADTs) let us represent data using "ANDs" and "ORs". For instance, a shape can be a rectangle _or_ a circle. A rectangle has a width _and_ a height. A circle has a radius.

```scala
sealed trait Shape
final case class Rectangle(width: Double, height: Double) extends Shape
final case class Circle(radius: Double) extends Shape
```

In ADT terminology, "AND" types (`Rectangle`, `Circle`) are referred to as _products_ and "OR" types (`Shape`) as _coproducts_.

Sealed traits and case classes are undoubtedly the most convenient encoding of ADTs in Scala. However, they aren’t the only encoding. We could instead have chosen `Tuple` and `Either` to encode products and coproducts, respectively:

```scala
type Rectangle = Tuple2[Double, Double]
type Circle = Double
type Shape = Either[Rectangle, Circle]
```

While the first example is more readable, the second is more generic. Any code that operates on a pair of doubles will also operate on the generic definition of our `Rectangle` type, and vice-versa.

Instead of `Tuple` and `Either`, Shapeless uses its own generic data types to represent generic products and coproducts.

## Generic product encodings
Shapeless uses a generic encoding for products called a heterogeneous list. An `HList` instance is either the empty list `HNil` or a pair `::[H, T]` where `H` is an arbitrary type and `T` is another `HList`.

Because the compiler knows the exact length of each `HList` instance, taking the head or tail of an empty list results in a compilation error.

## Generic coproduct encodings
In Shapeless coproducts take the form `A :+: B :+: C :+: CNil` meaning _“A or B or C”_, where `:+:` can be loosely interpreted as `Either`. `:+:` has two subtypes, `Inl` and `Inr`, corresponding to `Left` and `Right`. We can’t instantiate `CNil` or build a coproduct purely from instances of `Inr`. We always need exactly one `Inl` instance in a value.

```scala
import shapeless.{Coproduct, CNil, Inl, Irn, :+:}

case class Red()
case class Amber()
case class Green()

type Light = Red :+: Amber :+: Green :+: CNil

val red: Light = Inl(Red(())
val amber: Light = Inr(Inl(Amber()))
val green: Light = Inr(Inr(Inl(Green())))
```

## Generic type class
Shapeless provides a type class called `Generic` that lets us conveniently switch back and forth between a concrete ADT and its generic representation.

```scala
import shapeless.Generic

case class IceCream(
  name: String,
  numCherries: Int,
  inCone: Boolean
)

val iceCreamGen = Generic[IceCream]

val iceCream = IceCream("Sundae", 1, false)
// iceCream: IceCream = IceCream(Sundae, 1, false)
val repr = iceCreamGen.to(iceCream)
// repr: iceCreamGen.Repr = Sundae :: 1 :: false :: HNil
val iceCream2 = iceCreamGen.from(repr)
// iceCream2: IceCream = IceCream(Sundae, 1, false)
```

Instances of `Generic` have a `Repr` type member containing the type of their generic representation. In the example above, `iceCreamGen.Repr` would be `String :: Int :: Boolean :: HNil`.
