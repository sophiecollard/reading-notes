# Chapter 2 - Programming a Guessing Game
[Back to index](index.md)

## Terminology
The equivalent of Java's static methods in Rust are called *associated functions* and indicated by the `::` syntax (eg: `String::new`).

The equivalent of Scala's sum types are called *enumerations* or *enums*. Values from an enumeration's set of possible values are called *variants*. Eg: The `Result<T, E>` enumeration has two variants: `Ok(T)` and `Err(E)`.

## Syntax
`&` before a function argument indicates that the argument is a reference. Like variables, references are immutable by default and must be preceded by `&mut` to be mutable.
