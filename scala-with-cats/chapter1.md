# Chapter 1 - Introduction
[Back to index](index.md)

## Type classes
Type classes are a programming pattern originating in Haskell. They allow us to extend existing libraries with new functionality, without using traditional inheritance, and without altering the original library source code.

Type classes are interfaces or APIs that represent some functionality we want to implement. In Scala, they are implemented as traits with one or more type parameter(s).

Most of the functionality provided by Cats in implemented in the form of type classes.

## Type class interfaces
Type class interfaces are generic methods that accept instances of type classes as implicit parameters. For example, given a `JsonWriter` type class, we could have the following *interface object*:

```scala
object Json {
  def toJson[A](value: A)(implicit writer: JsonWriter[A]): Json = {
    writer.write(value)
  }
}
```

Alternatively, we could have the following *interface syntax*:

```scala
object JsonSyntax {
  implicit class JsonWriterOps[A](value: A) {
    def toJson(implicit writer: JsonWriter[A]): Json = {
      writer.write(value)
    }
  }
}
```

## Working with implicits

### Implicit scope
The compiler searches for candidate type classes instances in the implicit scope at the call site, which roughly consists of:
  * local or inherited definitions
  * imported definitions
  * definitions in the companion object of the type class or parameter type (in our case, the companion objects of `JsonWriter` and `A`, respectively)

If the compiler encounters multiple candidate instances, it fails with an *ambiguous implicit values error*.

### Recursive implicit resolution
We can construct implicit instances from other instances. For instance:

```scala
implicit def optionWriter[A](implicit writer: JsonWriter[A]): JsonWriter[Option[A]] = {
  new JsonWriter[A] {
    def write(value: Option[A]): Json = value match {
      case Some(a) => writer.write(a)
      case None    => Json.Null
    }
  }
}
```

When we invoke `Json.toJson(Option("A String"))`, the compiler will first discover the `JsonWriter[Option[A]]` instance, then recursively the `JsonWriter[String]` instance.

## The Eq type class
`Eq` is designed to implement type-safe equality and address shortcomings of Scala’s built-in `==` operator, which works for any pair of objects regardless of their type.

## Contravariance
Contravariance is useful for modeling types that represent processes, like `JsonWriter`.

```scala
trait JsonWriter[-A] {
  def write(value: A): Json
}
```

If `Circle` is a subtype of `Shape` (`Shape >: Circle`), then `JsonWriter[Shape]` must be a subtype of `JsonWriter[Circle]` (`JsonWriter[Circle] >: JsonWriter[Shape]`). This is because we can pass both `Shape` and `Circle` instance to a `JsonWriter[Shape]` instance, but only `Circle` instances to a `JsonWriter[Circle]`. In other words, we can use a `Circle` instance where a `Shape` is expected, but a `JsonWriter[Shape]` where a `JsonWriter[Circle]` is expected.
