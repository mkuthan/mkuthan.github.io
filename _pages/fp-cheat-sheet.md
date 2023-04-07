---
title: "FP cheat sheet"
permalink: /fp-cheat-sheet/
toc: true
---

As a software engineer with strong object oriented background
I need a functional programming design patterns cheat sheet.
Otherwise, I feel lost.

Credits:

* [Cats](https://typelevel.org/cats/index.html)
* [Scala with Cats](https://github.com/scalawithcats/scala-with-cats)
* [Scala exercises](https://www.scala-exercises.org/cats/)

## Semigroup

Associative binary operation.
The binary operation takes two elements of the set as input and produces a single output element of the set.

`combine(x, combine(y, z)) = combine(combine(x, y), z)`

Definition:

```scala
trait Semigroup[A] {
  def combine(x: A, y: A): A
}
```

Concatenate of two strings:

```scala
import cats.kernel.Semigroup

Semigroup[String].combine("A", "B") // "AB"
```

Add values of each key in the maps:

```scala
import cats.kernel.Semigroup

val map1 = Map("a" -> 1, "b" -> 2, "c" -> 3)
val map2 = Map("b" -> 3, "c" -> 4, "d" -> 5)

Semigroup[Map[String, Int]].combine(map1, map2) // Map(a -> 1, b -> 5, c -> 7, d -> 5)
```

Add values of each option if both are defined, or by returning the defined option, or else `None`:

```scala
import cats.kernel.Semigroup

val combine: (Option[Int], Option[Int]) => Option[Int] = Semigroup[Option[Int]].combine(_, _)

combine(Some(1), Some(2)) // Some(3)
combine(Some(1), None) // Some(1)
combine(None, Some(1)) // Some(1)
combine(None, None) // None
```

Add values of each try if both are successful, or by returning the first failed try, or else the second failed try:

```scala
import scala.util.{Try, Success, Failure}
import cats.kernel.Semigroup

val combine: (Try[Int], Try[Int]) => Try[Int] = Semigroup[Try[Int]].combine(_, _)

val f = Failure(new Exception())

combine(Success(1), Success(2)) // Success(3)
combine(Success(1), f) // Failure(java.lang.Exception)
combine(f, Success(1)) // Failure(java.lang.Exception)
combine(f, f) // Failure(java.lang.Exception)
```

## Monoid

A Monoid is a Semigroup that has an identity element.
An identity element is a value that when combined with any other value of the same type using Semigroup,
produces that other value as the result.

`combine(x, empty) = combine(empty, x) = x`

Definition:

```scala
trait Monoid[A] {
  def combine(x: A, y: A): A
  def empty: A
}
```

Sum of integers but use the Monoid's identity element to represent the missing value:

```scala
import cats.kernel.Monoid
import cats.syntax.semigroup._

val numbers = List(Some(1), None, Some(3), Some(4))

numbers.fold(Monoid[Option[Int]].empty)(_ |+| _) // Some(8)
```

## Functor

Generalizable map:

`fa.map(f).map(g) = fa.map(f.andThen(g))`

`fa.map(x => x) = fa`

![map](/assets/images/fp-cheat-sheet/functor-map.svg)

Definition:

```scala
trait Functor[F[_]] {
  def map[A, B](fa: F[A])(f: A => B): F[B]
}
```

For example:

```scala
def increment[F[_]](container: F[Int])(using functor: Functor[F]): F[Int] =
  functor.map(container)(_ + 1)
```

Or using extension method:

```scala
import cats.syntax.functor._
def increment[F[_]](container: F[Int])(using functor: Functor[F]): F[Int] =
  functor.map(container)(_ + 1)
```

### Contravariant Functor

```scala
def contramap[A, B](fa: F[A])(f: B => A): F[B]
```

![contramap](/assets/images/fp-cheat-sheet/functor-contramap.svg)

### Invariant Functor

Definition:

```scala
def imap[B](dec: A => B, enc: B => A): Codec[B]
```

![imap](/assets/images/fp-cheat-sheet/functor-imap.svg)

For example:

```scala
def longToDate: Long => Date = new Date(_)
def dateToLong: Date => Long = _.getTime

val semigroupDate: Semigroup[Date] =
  Semigroup[Long].imap(longToDate)(dateToLong)
```

## Monad

Mechanism for sequencing computations:

`pure(a).flatMap(func) == func(a)`

`m.flatMap(pure) == m`

`m.flatMap(f).flatMap(g) == m.flatMap(x => f(x).flatMap(g))`

Definition:

```scala
trait Monad[F[_]] {
  def pure[A](value: A): F[A]
  def flatMap[A, B](value: F[A])(func: A => F[B]): F[B]
}
```

![option flatmap](/assets/images/fp-cheat-sheet/monad-option-flatmap.svg)

### Identity Monad

Effect of having no effect:

```scala
type Id[A] = A
```

### Kleisli

Composition of functions that return a monadic value:

```scala
final case class Kleisli[F[_], A, B](run: A => F[B]) {
  def compose[Z](k: Kleisli[F, Z, A])(implicit F: FlatMap[F]): Kleisli[F, Z, B] =
    Kleisli[F, Z, B](z => k.run(z).flatMap(run))
}
```

## Semigroupal

Combine contexts:

`product(a, product(b, c)) == product(product(a, b), c)`

Definition:

```scala
trait Semigroupal[F[_]] {
  def product[A, B](fa: F[A], fb: F[B]): F[(A, B)]
}
```

## Applicative Functor

Definition:

```scala
trait Applicative[F[_]] extends Semigroupal[F] with Functor[F] {
  def ap[A, B](ff: F[A => B])(fa: F[A]): F[B]

  def pure[A](a: A): F[A]

  def map[A, B](fa: F[A])(f: A => B): F[B] = ap(pure(f))(fa)
}
```

For example:

```scala
val applicativeList = Applicative[List].pure(42)
```

Or using extension method:

```scala
import cats.syntax.applicative._
val applicativeList = 42.pure[List]
```

## Foldable

```scala
trait Foldable[F[_]] {
  def foldLeft[A, B](fa: F[A], b: B)(f: (A, B) => B): B
  def foldRight[A, B](fa: F[A], b: B)(f: (A, B) => B): B
}
```

![Fold List(1, 2, 3)](/assets/images/fp-cheat-sheet/fold-1-2-3.svg)

## Traverse

```scala
trait Traverse[F[_]] {
  def traverse[G[_]: Applicative, A, B]
      (inputs: F[A])(func: A => G[B]): G[F[B]]

  def sequence[G[_]: Applicative, B]
      (inputs: F[G[B]]): G[F[B]] =
    traverse(inputs)(identity)
}
```

For example, if `F` is `List` and `G` is `Option[Int]`:

```scala
def parseInt(s: String): Option[Int] = ???

List("1","2","3").traverse(parseInt) == Some(List(1,2,3))
List("1","a","3").traverse(parseInt) == None
```
