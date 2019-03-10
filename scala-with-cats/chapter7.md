# Chapter 7 - Foldable and Traverse
[Back to index](index.md)

## Foldable
The `Foldable` type class captures the `foldLeft` and `foldRight` methods we’re used to in sequences like `Lists`, `Vectors`, and `Streams`.

Depending on the operation we’re performing, the order in which we fold may be important. Because of this there are two standard variants of fold:
  * `foldLeft` traverses from *left* to *right* (start to finish);
  * `foldRight` traverses from *right* to *left* (finish to start).

The two variants are equivalent if our binary operation is associative.

A simplified implementation of Cats' `Foldable` is:

```scala
trait Foldable[F[_]] {
  def foldLeft[A, B](fa: F[A], b: B)(f: (B, A) => B): B
  def foldRight[A, B](fa: F[A], lb: Eval[B])(f: (A, Eval[B]) => Eval[B]): Eval[B]
}
```

Every method in `Foldable` is available in syntax form via the `cats.syntax.foldable` import. Remember that Scala will only use an instance of `Foldable` if the method isn’t explicitly available on the receiver. For example, the first code sample below will use the version of `foldLeft` defined on `List`, whereas the second generic code sample will use `Foldable`:

```scala
// Uses foldLeft defined on List
List(1, 2, 3).foldLeft(0)(_ + _)

// Uses foldLeft defined on Foldable
def sum[F[_]: Foldable](values: F[Int]): Int = {
  values.foldLeft(0)(_ + _)
}
```

### Stack safety of foldRight
Notice how `foldRight` uses `Eval` in order to ensure *stack safety*, even when the collection’s default definition of `foldRight` is not.

```scala
import cats.Eval
import cats.Foldable

def bigData = (1 to 100000).toStream

bigData.foldRight(0L)(_ + _)
// java.lang.StackOverflowError ...

import cats.instances.stream._

val eval: Eval[Long] = Foldable[Stream]
  .foldRight(bigData, Eval.now(0L)) { (n, eval) =>
    eval.map(_ + n)
  }

eval.value
// res0: Long = 5000050000
```

`Stream` is an exception in the Scala standard library for not having a stack-safe implementation of `foldRight`. Both `List` and `Vector` do.

### Usage
Here are a few examples of `Foldable` invocations:

```scala
import cats.Foldable
import cats.instances.list._ // for Foldable
import cats.instances.option._ // for Foldable

Foldable[List].foldLeft(List(1, 2, 3), 0)(_ + _)
// res0: Int = 6
Foldable[Option].foldLeft(Some(123), 10)(_ * _)
// res1: Int = 1230
Foldable[Option].foldLeft(Option.empty[Int], 10)(_ * _)
// res2: Int = 10
```
