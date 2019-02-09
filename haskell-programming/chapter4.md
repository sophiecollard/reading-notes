# Chapter 4
[back to index](index.md)

## Data types
The type constructor is the name of the type, capitalized. It is used at the type level (type signatures).

The data constructors are the values that inhabit the type they are defined in. They are used at the term level.

An example of a data declaration is:

```haskell
Data Bool = False | True
```

## Numeric types
All of the following numeric types have instances of the `Num` typeclass.
  * `Int` (also `Int8`, `Int16`): Fixed precision integer.
  * `Integer`: Also for integers, but arbitrarily large or small.
  * `Float`: Single-precision floating point number.
  * `Double`: Double-precision floating point number.
  * `Rational`: Carries two Integers - the numerator and denominator. Arbitrarily precise but not as efficient as `Scientific`.
  * `Scientific`: Represented using scientific notation. Stores the coefficient as an Integer and the exponent as an `Int`.

## Typeclasse constraints
A typeclass is a set of operations defined with respect to a polymorphic type.

In the type signature `(/) :: Fractional a => a -> a -> a`, `Fractional a =>` denotes a typeclass constraint.

We must be cautious when combining different instances of the `Num` typeclass. The example below will not compile because the length function returns an `Int`, which does not satisfy the typeclass constraint `Fractional a =>` typeclass constraint:

```haskell
6 / length [1, 2, 3]
```

This example below on the other hand will compile, thanks to `fromIntegral :: (Integral a, Num b) => a -> b`:

```haskell
6 / fromIntegral (length [1, 2, 3])
```

This will also work using `div :: Integral a => a -> a -> a`:

```haskell
div 6 (length [1, 2, 3])
```

## Pattern matching in anonymous function definitions
Haskell allows for pattern matching on anonymous functions parameters. These are valid function definitions:

```haskell
let head = (\(x : xs) -> x)
let fst = (\(a, b) -> a)
```
