# Chapter 5
[back to index](index.md)

## Type Signatures
A type signature might have multiple typeclass constraints:
```haskell
(Num a, Num b) => a -> b -> b
(Ord a, Num a) => a -> Ordering
```

Note that the `->` infix operator is right associative. Functions with an arity (number of parameters) greater than 1 are actually nested functions with arity 1.

Type signatures may have three kinds of types: concrete, constrained polymorphic, or parametrically polymorphic. Constrained (or ad-hoc) polymorphism in Haskell is implemented with typeclasses.

To declare a function type signature in the REPL, use the following syntax:

```haskell
let myAdd :: Num a => a -> a -> a; myAdd x y = x + y
```

## Uncurrying
Uncurrying is the opposite of currying. Concretely, un-currying the `(+)` operator would result in its type signature changing from `Num a => a -> a -> a` to Num `a => (a, a) -> a`. In short:
* Uncurried functions: one function, many arguments
* Curried functions: many functions, one argument each

Uncurried functions can easily be curried, as in the example below:
```haskell
let curry f a b = f(a, b)
:t fst
// fst :: (a, b) -> a
:t curry fst
// curry fst :: a -> b -> a
curry fst 1 2
// 1
```

Similarly for un-currying:

```haskell
let uncurry f (a, b) = f a b
:t (+)
// (+) :: Num a => a -> a -> a
:t uncurry (+)
// uncurry (+) :: Num a => (a, a) -> a
uncurry (+) (1, 2)
// 3
```
