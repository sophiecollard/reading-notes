# Chapter 2 - Monoids and Semigroups
[Back to index](index.md)

## Monoids
A monoid for a type `A` consists of:
  * an associative binary operation *combine* with type `(A, A) => A`
  * an identity element *empty* of type `A`

In Scala, this translates to:

```scala
trait Monoid[A] {
  def combine(x: A, y: A): A
  def empty: A
}
```

There may be more than one sensible implementation of the `Monoid` type class for any given type. For example `Monoid[Int]` could use *addition* or *multiplication*. `Monoid[Boolean]` could use any of *AND*, *OR*, *XOR* and *XNOR* operators. `Monoid[Set[A]]` could use *union*, *intersection* or even *diff*.

## Semigroups
A semigroup only has an associative binary operation *combine*:

```scala
trait Semigroup[A] {
  def combine(x: A, y: A): A
}
```

Using inheritance, we can redefine `Monoid` as follows:

```scala
trait Monoid[A] extends Semigroup[A] {
  def empty: A
}
```
