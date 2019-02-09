# CHAPTER 3: Automatically deriving type class instances
[back to index](index.md)

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

The commonly accepted idiomatic style for type class definitions includes a companion object containing some standard methods:

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
Shapeless can be used to derive type class instances, using the following tricks:
* If we have type class instances for the head and tail of an `HList`, we can derive an instance for the whole `HList`.
* If we have a case class `A`, an instance of `Generic[A]` and a type class instance for the generic's `Repr`, we can combine the three of them to create an instance for `A`.

In the `IceCream` example from [chapter 2](chapter2.md):
1. `IceCream` has a generic `Repr` of type `String :: Int :: Boolean :: HNil`.
2. The `Repr` is made up of a `String`, an `Int` and a `Boolean`. If we have `CsvEncoder` instances for each of these, we can derive a `CsvEncoder[String :: Int :: Boolean :: HNil]` instance.
3. We can derive `CsvEncoder[IceCream]` from `CsvEncoder[String :: Int :: Boolean :: HNil]`.

### Deriving type class instances for HList
In this section, a `CsvEncoder` instance is derived for the generic representation of our `IceCream` type `String :: Int :: Boolean :: HNil` in 3 steps.

#### Step 1
Let's define `CsvEncoder` for the three primitive types of interest:

```scala
implicit val stringCsvEncoder: CsvEncoder[String] = CsvEncoder.instance { string =>
  List(string)
}

implicit val intCsvEncoder: CsvEncoder[Int] = CsvEncoder.instance { int =>
  List(int.toString)
}

implicit val booleanCsvEncoder: CsvEncoder[Boolean] = CsvEncoder.instance { boolean =>
  List(boolean.toString)
}
```

#### Step 2
We can combine the primitive encoders defined above to define an encoder for an instance of `String :: Int :: Boolean :: HNil`.

```scala
import shapeless.{HList, HNil, ::}

implicit val hNilCsvEncoder: CsvEncoder[HNil] = CsvEncoder.instance { _ => Nil }

implicit def hListCsvEncoder[H, T <: HList](
  implicit
  hCsvEncoder: CsvEncoder[H],
  tCsvEncoder: CsvEncoder[T]
): CsvEncoder[H :: T] = CsvEncoder.instance { case h :: t =>
  hCsvEncoder.encode(h) ++ tCsvEncoder.encode(t)
}
```

#### Step 3
Taken together, the 5 instances above let us summon a `CsvEncoder` instance for any `HList` involving `String`, `Int` and `Boolean`:

```scala
// Note: The example provided in the book using simply `implicitly` fails to compile
val reprCsvEncoder: CsvEncoder[String :: Int :: Boolean :: HNil] = 
  implicitly[CsvEncoder[String :: Int :: Boolean :: HNil]]

reprCsvEncoder.encode("abc" :: 1 :: false :: HNil)
// res0: List[String] = List(abc, 1, false)
```

### Deriving type class instances for concrete products
Combining our derivation rules for `HList` with a `Generic[IceCream]` instance, we can derive a `CsvEncoder[IceCream]` instance.

```scala
object IceCream {
  import shapeless.Generic

  implicit val csvEncoder: CsvEncoder[IceCream] = {
    val generic = Generic[IceCream]
    val genericCsvEncoder = CsvEncoder[generic.Repr]
    CsvEncoder.instance { iceCream =>
      genericCsvEncoder.encode(generic.to(iceCream))
    }
  }
}
```

We can test our encoder as follows:

```scala
val iceCream = IceCream("Sundae", 1, false)
val iceCreamCsvEncoder: CsvEncoder[IceCream] = implicitly[CsvEncoder[IceCream]]
iceCreamCsvEncoder.encode(iceCream)
```

The example above is specific to our `IceCream` type. Ideally, we'd like to derive `CsvEncoder` instances for any type with a `Generic` instance and a matching `CsvEncoder` for that instance.

This can be achieved with:

```scala
implicit def genericCsvEncoder[A, R](
  implicit
  //gen: Generic[A] { type Repr = R }, // trick to solve scoping issue
  gen: Generic.Aux[A, R], // see "Aux type aliases" section below
  csvEncoder: CsvEncoder[R]
): CsvEncoder[A] = CsvEncoder.instance { a =>
  csvEncoder.encode(gen.to(a))
}
```

We can test this code in the repl:

```scala
import shapeless.{Generic, HList, HNil, ::}

val iceCream = IceCream("Sundae", 1, false)
implicit val genIceCream = Generic[IceCream]

// check that we have the HList encoder correctly imported into our scope
implicitly[CsvEncoder[String :: Int :: Boolean :: HNil]]
// res0: typeclasses.CsvEncoder[String :: Int :: Boolean :: shapeless.HNil] = typeclasses.CsvEncoder$$anon$1@b57ca42

val iceCreamCsvEncoder = implicitly[CsvEncoder[IceCream]]

iceCreamCsvEncoder.encode(iceCream)
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
When things go wrong, the compiler messages can be quite useless. This usually happens when the compiler can't find an instance of `Generic` or when it can't derive a `CsvEncoder` instance for the corresponding `HList`.  This normally happens because one of the fields in our ADT does not have a `CsvEncoder` instance in scope.

## Deriving coproduct type class instances
_
