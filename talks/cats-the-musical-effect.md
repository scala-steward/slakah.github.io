class: center, middle, inverse

# Cats the ~~Musical~~ Effect

James Collier @JRCCollier

![Cats Logo](./assets/cats-logo.png)

---

## Aim of this talk

Quick intro, there are better docs elsewhere. Go look at them!

Focus on practical application, and issues I've personally encountered.

N.B. `IO` and referential transparency is a well trodden topic, so this talk will mainly focus on other areas of cats-effect.

---
class: inverse

## Hopefully less scary than 😱

![Scary Cats](./assets/cats-musical-film.png)

*The cursed trailer of cats the musical film*

---

## Getting Started

`build.sbt`:

```scala
libraryDependencies += "org.typelevel" %% "cats-effect" % "2.0.0"
```

All the imports you are likely to need:

```scala mdoc
import cats.effect._
import cats.implicits._
```

Will need an implicit `ContextShift` and `Timer`.

Prefer `IOApp` (will cover that later), manual creation is possible
as so:

```scala mdoc:silent
import scala.concurrent._

implicit val cs: ContextShift[IO] = IO.contextShift(ExecutionContext.global)
implicit val timer: Timer[IO] = IO.timer(ExecutionContext.global)
```

---

## IO

A way to encode an effect.

```scala mdoc
// effect is to write "hello" to stdout
val helloIO = IO(println("hello"))

```

`"hello"` will not be printed until `helloIO` is ran.

```scala mdoc
helloIO.unsafeRunSync()
```

---

## Triggering effects

Anything which isn't pure (side effecting) in cats-effect will
state that it is `unsafe`.

API is much simpler and more explicit than the Future alternative:

```scala mdoc:silent
import ExecutionContext.Implicits.global
import scala.concurrent.duration._
```

```scala mdoc
Await.result(Future(println("wibble")), 1.seconds)
```

vs

```scala mdoc
IO(println("wibble")).unsafeRunSync()
```

---

## Reasons to switch

Ditch scala `Future`s use `IO`.

---

## Semantic blocking

sleep is handled without blocking a thread, sleep is scheduled
using a timer.

Modelling delays using `IO` is trivial.

```scala mdoc
val io = for {
  _ <- IO(println("first"))
  _ <- IO.sleep(1.second)
  _ <- IO(println("second"))
} yield ()

io.unsafeRunSync()
```

This is much easier to read than the comparable implementation using akka after:

```scala
val scheduler = ???

for {
  _ <- Future(println("first"))
  _ <- akka.pattern.after(1.second, scheduler)(_ =>
    Future(println("second"))
  )
} yield ()
```

---

## Currency primitives

cats-effect implements a number of concurrency primitives which are also non blocking.

`Deferred` - similar to `scala.concurrent.Promise`
* `MVar` - concurrent queue
* `Ref` - functional `java.util.concurrent.atomic.AtomicReference`
* `Semaphore` - functional semaphore.

---

## Cancellation

`scala.concurrent.Future` doesn't support cancellation.

It's quite difficult to prevent `second` being printed.

```scala mdoc:silent
val fut = for {
  _ <- Future(println("first"))
  _ = Thread.sleep(10000)
  _ <- Future(println("second"))
} yield ()
```

Even Java supports cancellation, see: `java.util.concurrent.CompletableFuture`.

---

## Timeout + cancellation example

Timeouts are trivial in cats-effect.

```scala mdoc:silent
val printingIO = for {
  _ <- IO(println("first"))
  _ <- IO.sleep(1.second)
  _ <- IO(println("second"))
} yield ()
printingIO.timeout(0.5.seconds)
```

---

## Go to the Races

![Horse race](./assets/horse-race.png)

`IO.race` is a way to pit two `IO`s against each other, only the fastest
wins!

---

## IO Race example

Race some `IO`s!

```scala mdoc:silent
def startHorse(raceTime: FiniteDuration, horseName: String) = for {
  _ <- IO.sleep(raceTime)
  _ <- IO(println(s"$horseName finished!"))
} yield horseName

def letTheRaceBegin() = for {
  _ <- IO(println("starting pistol fired!"))
  raceResult <- IO.race( // Either[String, String]
    startHorse(1.seconds, "Red Rum"),
    startHorse(3.second, "Seabiscuit")
  )
  winningHorse = raceResult.merge
  _ <- IO(println(s"$winningHorse won!"))
} yield ()
```

```scala mdoc
letTheRaceBegin().unsafeRunSync()
```

---

## Cancellation is very handy

Means that no unnecessary work is carried out, as an example:

If you were to handle a web request, which executes for a longer
period than the configured request timeout. Using `IO`
no extra work would be scheduled after the timeout occurred.

## Can't stop me!

An `IO` can also be made to become uncancelable using `.uncancelable`.

---

## Data Types

Let's look at some of the data types now!

---

## `IOApp`

Utility to help wire in an `IO` into your main class.

```scala mdoc:silent

object Main extends IOApp {

  def run(args: List[String]): IO[ExitCode] =
    args.headOption match {
      case Some(name) =>
        IO(println(s"Hello, $name.")).as(ExitCode.Success)
      case None =>
        IO(System.err.println("Usage: MyApp name")).as(ExitCode(2))
    }
}
```
*example from [https://typelevel.org/cats-effect]()*

---

## `cats.effect.Resource`

Elegant way to safely acquire and release resources.

Things which should be considered resources include:

* File handles
* Connections
* Thread pools
* AWS SDK Clients

---

## `Resource` examples

### pre Scala 2.13

```scala mdoc:silent
import scala.io._

val configSource: BufferedSource = Source.fromFile("talks/test-conf.json")
try {
  // do stuff with config source
} finally {
  configSource.close()
}
```

### Scala 2.13 onwards

```scala
import scala.util.Using

Using.resource(Source.fromFile("talks/test-conf.json")) { configSource =>
  // do stuff with config source
}
```

---

## Resource

```scala mdoc
import scala.io._

def readFile(path: String): Resource[IO, BufferedSource] = Resource
  .fromAutoCloseable(IO(Source.fromFile(path)))

val readConfig = readFile("talks/test-conf.json").use(lines => IO(lines.mkString))
readConfig.unsafeRunSync()
```

---

## Combining Resources

Resources can be combined using for comprehension or `.tupled`.

```scala mdoc:silent
val readAllConfs = (readFile("talks/app-conf.json"), readFile("talks/test-conf.json"))
  .tupled
  .use { case (lines1, lines2) =>
    IO((lines1 ++ lines2).mkString)
  }
```

```scala mdoc
readAllConfs.unsafeRunSync()
```

---

## Blocker

Sometimes it's just easier to block. A lot of useful Java libraries block, so how do we handle this?

![Concurrency Thread Pools](./assets/concurrency-thread-pools.png)

*from [@impurepics](https://twitter.com/impurepics) in cats-effect docs*

---

## How to use?

```scala
// load blocker resource
val blocker: Blocker = ???
val blockingOp: IO[Unit] = ???
blocker.blockOn(blockingOp)
```

---

## Blocker on the JVM

`Blocker[IO]` is effectively cached thread pool.

```scala
/**
  * Creates a blocker that is backed by a cached thread pool.
  */
def apply[F[_]](implicit F: Sync[F]): Resource[F, Blocker] =
  fromExecutorService(F.delay(Executors.newCachedThreadPool(new ThreadFactory {
    def newThread(r: Runnable) = {
      val t = new Thread(r, "cats-effect-blocker")
      t.setDaemon(true)
      t
    }
  })))
```
*from cats-effect source code*

---

## InfluxDB client example

InfluxDB Java Client is blocking, let's use `Blocker[IO]`!

connect to InfluxDB:
![InfluxDB connect](./assets/influxdb-connect.png)

Blocking write:
![InfluxDB write](./assets/influxdb-write.png)

---

## Timer

Handles sleeping, and getting the current time.

```scala
trait Timer[F[_]] {
  def clock: Clock[F]

  def sleep(duration: FiniteDuration): F[Unit]
}
```
*from cats-effect source code*

For the JVM `Clock[F]`, is an interface over `System.currentTimeMillis()` and `System.nanoTime()`.

---

## Testing sleeps quickly

Produce fake playback session events are published with delays in between.

Difficult to test due to sleeps in between events, modelled playback sessions can take 30min to complete.

How do we test this?

---

## IOTestTimer

```scala mdoc
import java.util.concurrent.TimeUnit
import java.time.Instant
import cats.effect.concurrent.Ref

class IOTestTimer private (currentTime: Ref[IO, Instant]) extends Timer[IO] {

  def clock: Clock[IO] = new Clock[IO] {
    override def realTime(unit: TimeUnit): IO[Long] =
      currentTime.get.map(curr => unit.convert(curr.toEpochMilli, TimeUnit.MILLISECONDS))
    override def monotonic(unit: TimeUnit): IO[Long] = realTime(unit)
  }

  def sleep(duration: FiniteDuration): IO[Unit] =
    currentTime.update(_.plusMillis(duration.toMillis))
}

object IOTestTimer {
  def fromNow(): IO[Timer[IO]] =
    for {
      now <- IO(Instant.now())
      ref <- Ref.of[IO, Instant](now)
    } yield new IOTestTimer(ref)
}
```

---

## Using IOTestTimer

TODO: add example

---

## Further reading and links

* [Intro to cats-effect](https://daenyth.github.io/intro-cats-effect/) by [@Daenyth](https://twitter.com/Daenyth)
* [cats-effect docs](https://typelevel.org/cats-effect/)
* [@impurepics](https://twitter.com/impurepics) providing some graphics for the community

## Thanks!

## Questions?
