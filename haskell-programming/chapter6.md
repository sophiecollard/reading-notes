# Chapter 6

## Typeclass Derivation
Typeclasses which can automatically be derived are `Eq`, `Ord`, `Enum`, `Bounded`, `Read` and `Show`, though there are some constraints on doing that.

## Non-exhaustive Match Compiler Warning
To enable warnings for non-exhaustive matches in the REPL, use:
```haskell
:set Wall
```
