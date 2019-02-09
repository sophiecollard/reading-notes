# Chapter 1
[back to index](index.md)

## Associativity
The expression `a [op] b [op] c` get parenthesized as `(a [op] b) [op] c` if `[op]` is _left-associative_, and as `a [op] (b [op] c)` if it is _right-associative_. The operator `[op]` is said to be _associative_ if both parenthesizing (left or right associativity) yield the same result.

## Lambda calculus
Applications in lambda calculus are left-associative. So `(λx.x)(λy.y)z` will first evaluate to `(λy.y)z` and then to `z`.

In the expression `λx.xy`, `x` is a bound variable while `y` is a free variable. A _combinator_ is a lambda term with no free variables.

When evaluated, expressions reduce to values, a process known as _beta reduction_.
