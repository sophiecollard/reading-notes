# Chapter 1 - Introduction
[Back to index](index.md)

## Motivations for Generic Programming
As programmers, we sometimes find ourselves wanting to exploit similarities between types. Consider the following definitions:

```scala
case class Employee(name: String, id: Int, isManager: Boolean)

case class IceCream(name: String, numCherries: Int, inCone: Boolean)
```

While representing different types, both case classes contain three fields of the same type. Suppose we wanted to serialize instances of those classes to a CSV file. Instead of defining serialization methods specific to each class, we'd like to exploit the similarities between them and define a single *generic* serialization method.

Libraries like Shapeless conveniently let us convert specific types to generic ones that can then be manipulated with common code.

### Example
In the example below, we define a single method capable of converting `Generic` instances of `Employee` and `IceCream` to `List[String]` for persisting to a CSV file.

```scala
import shapeless._

val genericEmployee = Generic[Employee].to(Employee("Dave", 123, false))
val genericIceCream = Generic[IceCream].to(IceCream("Sundae", 1, false))

def genericSerialize(gen: String :: Int :: Boolean :: HNil): List[String] =
List(gen(0), gen(1).toString, gen(2).toString)

genericSerialize(genericEmployee)
// res0: List[String] = List(Dave, 123, false)

genericSerialize(genericIceCream)
// res1: List[String] = List(Sundae, 1, false)
```
