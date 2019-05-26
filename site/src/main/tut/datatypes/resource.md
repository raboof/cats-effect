---
layout: docsplus
title:  "Resource"
number: 14
source: "core/shared/src/main/scala/cats/effect/Resource.scala"
scaladoc: "#cats.effect.Resource"
---

Effectfully allocates and releases a resource. Forms a `MonadError` on the resource type when the effect type has a `Bracket` instance.

The [Acquiring and releasing `Resource`s](../tutorial/tutorial.html#acquiring-and-releasing-resources) section of the tutorial provides some additional context and examples regarding `Resource`.

```tut:silent
import cats.effect.Bracket

abstract class Resource[F[_], A] {
  def use[B](f: A => F[B])(implicit F: Bracket[F, Throwable]): F[B]
}
```

Nested resources are released in reverse order of acquisition. Outer resources are released even if an inner use or release fails.

You can lift any `F[A]` with an `Applicative` instance into a `Resource[F, A]` with a no-op release via `Resource.liftF`:

```
import cats.effect.{IO, Resource}
import cats.implicits._

val greet: String => IO[Unit] = x => IO(println("Hello " ++ x))

Resource.liftF(IO.pure("World")).use(greet).unsafeRunSync
```

Note that resource acquisition cannot be canceled, so if you lift a long running `F[A]` into `Resource` and cancel the `use` call while it runs,
the program will wait until that `F[A]` completes.

For example, if you do this:

```
//latch to make sure we cancel the `use` after it's actually started
Deferred[IO, Unit].flatMap { promise =>
  Resource.liftF(promise.complete(()) >> IO.never: IO[Unit])
    .use(_ => IO.unit) race promise.get
}
```

the program will never terminate.

Moreover it's possible to apply further effects to the wrapped resource without leaving the `Resource` context via `evalMap`:

```
import cats.effect.{IO, Resource}
import cats.implicits._

val acquire: IO[String] = IO(println("Acquire cats...")) *> IO("cats")
val release: String => IO[Unit] = x => IO(println("...release everything"))
val addDogs: String => IO[String] = x =>
  IO(println("...more animals...")) *> IO.pure(x ++ " and dogs")
val report: String => IO[String] = x =>
  IO(println("...produce weather report...")) *> IO("It's raining " ++ x)

Resource.make(acquire)(release).evalMap(addDogs).use(report).unsafeRunSync

```
### Example

```tut:silent
import cats.effect.{IO, Resource}
import cats.implicits._

def mkResource(s: String) = {
  val acquire = IO(println(s"Acquiring $s")) *> IO.pure(s)

  def release(s: String) = IO(println(s"Releasing $s"))

  Resource.make(acquire)(release)
}

val r = for {
  outer <- mkResource("outer")
  inner <- mkResource("inner")
} yield (outer, inner)

r.use { case (a, b) => IO(println(s"Using $a and $b")) }.unsafeRunSync
```

If using an AutoCloseable create a resource without the need to specify how to close.

### Examples

#### With `scala.io.Source`

```tut:silent
import cats.effect._

val acquire = IO {
  scala.io.Source.fromString("Hello world")
}

Resource.fromAutoCloseable(acquire).use(source => IO(println(source.mkString))).unsafeRunSync()
```

#### With `java.io` using IO

```tut:silent
import java.io._
import collection.JavaConverters._
import cats.effect._

def readAllLines(bufferedReader: BufferedReader): IO[List[String]] = IO {
  bufferedReader.lines().iterator().asScala.toList
}

def reader(file: File): Resource[IO, BufferedReader] =
  Resource.fromAutoCloseable(IO {
        new BufferedReader(new FileReader(file))
      }
  )

def readLinesFromFile(file: File): IO[List[String]] = {
    reader(file).use(readAllLines)
}
```

#### A `java.io` example agnostic of the effect type

```tut:silent
import java.io._
import cats.effect._

def reader[F[_]](file: File)(implicit F: Sync[F]): Resource[F, BufferedReader] =
  Resource.fromAutoCloseable(F.delay {
    new BufferedReader(new FileReader(file))
  })

def dumpResource[F[_]](res: Resource[F, BufferedReader])(implicit F: Sync[F]): F[Unit] = {
  def loop(in: BufferedReader): F[Unit] =
    F.suspend {
      val line = in.readLine()
      if (line != null) {
        System.out.println(line)
        loop(in)
      } else {
        F.unit
      }
    }
  res.use(loop)
}

def dumpFile[F[_]](file: File)(implicit F: Sync[F]): F[Unit] =
  dumpResource(reader(file))

```