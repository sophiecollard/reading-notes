# Chapter 3 - Automatic Derivation of Type Class Instances
[Back to index](index.md)

## Type classes recap
A type class is a parameterised trait representing some sort of general functionality that we would like to apply to a wide range of types. We implement instances of our type classes for each concrete type we care about. For instance:

```scala
// type class
trait CsvEncoder[A] {
  def encode(value: A): List[String]
}

// concrete type
final case class Employee(name: String, id: Int, isManager: Boolean)

// CsvEncoder[Employee] instance
implicit val employeeCsvEncoder: CsvEncoder[Employee] = {
  new CsvEncoder[Employee] = {
    def encode(employee: Employee): List[String] = {
      List(employee.name, employee.id.toString, employee.isManager.toString)
    }
  }  
}
```

Idiomatic style for type class definitions includes a companion object containing some standard methods:

```scala
object CsvEncoder {
  // "Summoner" or "Materializer" method
  def apply[A](implicit ev: CsvEncoder[A]): CsvEncoder[A] = ev

  // "Constructor" method
  def instance[A](f: A => List[String]): CsvEncoder[A] = {
    new CsvEncoder[A] {
      def encode(value: A): List[String] = f(value)
    }
  }
}
```

## Deriving product type class instances
Shapeless can be used to derive type class instances for products using the following tricks:
* If we have type class instances for the head and tail of an `HList`, we can derive an instance for the whole `HList`.
* If we have a case class `A`, an instance of `Generic[A]` and a type class instance for `Generic[A].Repr`, we can combine the three of them to create a type class instance for `A`.

In the next two sections, we look type class instance derivation, first for `HList`s then for concrete product types.

### For HList
In this section, we derive a `CsvEncoder` instance for the generic representation of our `IceCream` type from [chapter 2](chapter2.md), `Generic[IceCream].Repr = String :: Int :: Boolean :: HNil`. This can be achieved in 3 steps:

#### Step 1
We start by defining `CsvEncoder` instances for the three primitive types of interest.

```scala
implicit val stringCsvEncoder: CsvEncoder[String] =
  CsvEncoder.instance { string =>
    List(string)
  }

implicit val intCsvEncoder: CsvEncoder[Int] =
  CsvEncoder.instance { int =>
    List(int.toString)
  }

implicit val booleanCsvEncoder: CsvEncoder[Boolean] =
  CsvEncoder.instance { boolean =>
    List(boolean.toString)
  }
```

#### Step 2
The primitive encoders defined above can then be combined to define an encoder for a `String :: Int :: Boolean :: HNil` instance.

```scala
import shapeless.{HList, HNil, ::}

implicit val hNilCsvEncoder: CsvEncoder[HNil] =
  CsvEncoder.instance(_ => Nil)

implicit def hListCsvEncoder[H, T <: HList](
  implicit
  hCsvEncoder: CsvEncoder[H],
  tCsvEncoder: CsvEncoder[T]
): CsvEncoder[H :: T] =
  CsvEncoder.instance { case h :: t =>
    hCsvEncoder.encode(h) ++ tCsvEncoder.encode(t)
  }
```

#### Step 3
Taken together, the 5 instances defined in steps 1 and 2 let us summon a `CsvEncoder` instance for any `HList` involving `String`, `Int` and `Boolean`:

```scala
// Note: The example provided in the book using simply `implicitly` fails to compile
val reprCsvEncoder: CsvEncoder[String :: Int :: Boolean :: HNil] = 
  implicitly[CsvEncoder[String :: Int :: Boolean :: HNil]]

reprCsvEncoder.encode("Sundae" :: 1 :: false :: HNil)
// res0: List[String] = List(Sundae, 1, false)
```

### For concrete products
Combining our derivation rules for `HList` with a `Generic[IceCream]` instance, we can derive a `CsvEncoder[IceCream]` instance.

```scala
import shapeless.Generic

object IceCream {

  implicit val csvEncoder: CsvEncoder[IceCream] = {
    val generic = Generic[IceCream]
    val genericCsvEncoder = CsvEncoder[generic.Repr]
    CsvEncoder.instance { iceCream =>
      genericCsvEncoder.encode(generic.to(iceCream))
    }
  }

}
```

We can test our `IceCream` encoder with:

```scala
val iceCream = IceCream("Sundae", 1, false)
implicitly[CsvEncoder[IceCream]].encode(iceCream)
// res0: List[String] = List(Sundae, 1, false)
```

The example above is specific to the `IceCream` type. Ideally, we'd like to derive `CsvEncoder` instances for any type with a `Generic` instance and a matching `CsvEncoder` for that instance.

This can be achieved with the following method:

```scala
import shapeless.Generic

implicit def genericCsvEncoder[A, R](
  implicit
  // See "Aux type aliases" section below
  // Alternatively, we could have solved the scoping issue with:
  // gen: Generic[A] { type Repr = R },
  gen: Generic.Aux[A, R],
  csvEncoder: CsvEncoder[R]
): CsvEncoder[A] =
  CsvEncoder.instance { a =>
    csvEncoder.encode(gen.to(a))
  }
```

We can test our new generic encoder with:

```scala
import shapeless.{Generic, HNil, ::}

val iceCream = IceCream("Sundae", 1, false)
implicit val genIceCream = Generic[IceCream]

// Just checking that we have the HList encoder correctly imported into our scope
implicitly[CsvEncoder[String :: Int :: Boolean :: HNil]]
// res0: CsvEncoder[String :: Int :: Boolean :: shapeless.HNil] = CsvEncoder$$anon$1@46fcc710

implicitly[CsvEncoder[IceCream]].encode(iceCream)
// res1: List[String] = List(Sundae, 1, false)
```

### Aux type aliases
Type refinements like `Generic[A] { type Repr = R }` are verbose and difficult to read, so shapeless provides a type alias `Generic.Aux` to rephrase the type member as a type parameter:

```scala
package shapeless

object Generic {
  type Aux[A, R] = Generic[A] { type Repr = R }
}
```

### Compiler errors
When things go wrong, the compiler messages can be quite useless. It may be the case that the compiler can't find an instance of `Generic` or that it can't derive a `CsvEncoder` instance for the corresponding `HList`.  This normally happens because one of the fields in our ADT does not have a `CsvEncoder` instance in scope.

## Deriving coproduct type class instances
We return to our `Shape` ADT from [chapter2](chapter2.md) and attempt to derive a `CsvEncoder` instance for it.

### For generic coproducts
Using the same principles as for `HList`, we define generic `CsvEncoder` instances for `:+:` and `CNil`. Our generic coproduct encoder differs from our `HList` encoder in two ways:
  * Because `CNil` cannot be instantiated, our `CsvEncoder[CNil]` instance will never be invoked and therefore can afford to throw an exception.
  * Because coproducts are _disjuctions_ of types, our encoder must pattern match on the two subtypes of `:+:` - `Inl` and `Inr`.

```scala
import shapeless.{Coproduct, CNil, Inl, Inr, :+:}

implicit val cnilCsvEncoder: CsvEncoder[CNil] =
  CsvEncoder.instance(_ => throw new RuntimeException("Impossible to encode CNil!"))

implicit def coproductCsvEncoder[H, T <: Coproduct](
  implicit
  hCsvEncoder: CsvEncoder[H],
  tCsvEncoder: CsvEncoder[T]
): CsvEncoder[H :+: T] =
  CsvEncoder.instance {
    case Inl(h) => hCsvEncoder.encode(h)
    case Inr(t) => tCsvEncoder.encode(t)
  }
```

We can test our generic coproduct encoder with:

```scala
import shapeless.{CNil, Inl, :+:}

val rectangle = Inl(Rectangle(4.0, 3.0))
// res0: Inl[Rectangle, Nothing] = Inl(Rectangle(4.0, 3.0))

implicitly[CsvEncoder[Rectangle :+: Circle :+: CNil]].encode(rectangle)
// res1: List[String] = List(4.0, 3.0)
```

### For concrete coproducts
In order to derive a `CsvEncoder` instance for our `Shape` ADT, we must remember to define a `CsvEncoder[Double]` instance.

```scala
implicit val doubleCsvEncoder: CsvEncoder[Double] =
  CsvEncoder.instance { double =>
    List(double.toString)
  }
```

We can test our concrete coproduct encoder with:

```scala
implicitly[CsvEncoder[Rectangle]].encode(Rectangle(4.0, 3.0))
// res0: List[String] = List(4.0, 3.0)

implicitly[CsvEncoder[Shape]].encode(Rectangle(4.0, 3.0))
// res1: List[String] = List(4.0, 3.0)
```

## Deriving type class instances for recursive types
Consider the following `Tree` ADT:

```scala
sealed trait Tree[A]
final case class Branch[A](left: Tree[A], right: Tree[A]) extends Tree[A]
final case class Leaf[A](value: A) extends Tree[A]
```

### Implicit divergence
Implicit resolution is a search process. The compiler uses heuristics to determine whether it is converging on a solution. One such heuristic is designed to avoid infinite loops. If the compiler encounters the same target type more than once in a particular branch of search, it gives up on that branch and moves onto another. Another heuristic is designed to stop the search process when a branch appears to be diverging. A search branch is considered to be divergent if the complexity of the type parameter for a given type constructor increases from one iteration to the next.

Looking at the implicit resolution process for a `CsvEncoder[Tree[Int]]` instance, we can see it return to the same target type it started with:

```scala
CsvEncoder[Tree[Int]]
CsvEncoder[Branch[Int] :+: Leaf[Int] :+: CNil]
CsvEncoder[Branch[Int]]
CsvEncoder[Tree[Int] :: Tree[Int] :: HNil]
CsvEncoder[Tree[Int]]
```

Seeing multiple occurrences of `CsvEncoder[Tree[Int]]`, the compiler will think it got stuck in an infinite loop and give up on trying to implicity resolve the `CsvEncoder[Tree[Int]]` instance.

Now, consider resolving a `CsvEncoder` instance for the following `Foo` case class:

```scala
final case class Bar(baz: Int, qux: String)
final case class Foo(bar: Bar)
```

The implicit resolution will start as follows:

```scala
CsvEncoder[Foo]
CsvEncoder[Bar :: HNil]
CsvEncoder[Bar]
CsvEncoder[Int :: String :: HNil]
```

Again, seeing the complexity of the type parameter `T` for the `::[H, T]` type constructor increase from the 2nd to the 4th iteration, the compiler will think the search branch is divergent and give up.

### Lazy
To work around implicit divergence, Shapeless provides a type called `Lazy` which (1.) guards against the aforementioned over-defensive convergence heuristics at compile time and (2.) defers evaluation of implicit parameters at runtime.

`Lazy` is wrapped around specific implicit parameters. As a rule of thumb, those include the head of any `HList` or `Coproduct` as well as type class instances for the `Repr` parameter of any `Generic` instance.

```scala
import shapeless.{Coproduct, Generic, HList, Inl, Inr, Lazy, ::}

implicit def hListCsvEncoder[H, T <: HList](
  implicit
  hCsvEncoder: Lazy[CsvEncoder[H]], // wrap in Lazy
  tCsvEncoder: CsvEncoder[T]
): CsvEncoder[H :: T] =
  CsvEncoder.instance { case h :: t =>
    hCsvEncoder.value.encode(h) ++ tCsvEncoder.encode(t)
  }

implicit def coproductCsvEncoder[H, T <: Coproduct](
  implicit
  hCsvEncoder: Lazy[CsvEncoder[H]], // wrap in Lazy
  tCsvEncoder: CsvEncoder[T]
): CsvEncoder[H :+: T] =
  CsvEncoder.instance {
    case Inl(h) => hCsvEncoder.value.encode(h)
    case Inr(t) => tCsvEncoder.encode(t)
  }

implicit def genericCsvEncoder[A, R](
  implicit
  gen: Generic.Aux[A, R],
  csvEncoder: Lazy[CsvEncoder[R]] // wrap in Lazy
): CsvEncoder[A] =
  CsvEncoder.instance { a =>
    csvEncoder.value.encode(gen.to(a))
  }
```

We can test our encoder on a complex/recursive structure like `Tree` with:

```scala
implicitly[CsvEncoder[Tree[Int]]].encode(
  Branch(
    left = Leaf(4),
    right = Branch(
      left = Leaf(1),
      right = Leaf(2)
    )
  )
)
// res0: List[String] = List(4, 1, 2)
```
