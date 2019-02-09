# Chapter 7
[back to index](index.md)

## Newtype
`newtype` is a special type of data declaration in that in permits only one constructor and only one field.

```haskell
newtype N = N Int
```

## GHCI Browsing
To see a list of type signatures and functions loaded from a module ModuleX, use:

```haskell
:browse ModuleX
```
