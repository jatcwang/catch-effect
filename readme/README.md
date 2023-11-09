# Catch effect <a href="https://typelevel.org/cats/"><img src="https://typelevel.org/cats/img/cats-badge.svg" height="40px" align="right" alt="Cats friendly" /></a>
Catch effect is a small library for piling effectful algebras on top of eachother.

A small set of structures are introduced that allow construction of algebras akin to MTL, but without monad transformers and lifting effects.

Catch effect is built on top of [Vault](https://github.com/typelevel/vault), which is a map that can hold values of different types.

## Context
Context can create instances of `Local`, that represent running an effect given some input.
The MTL counterpart of `Context` is `Kleisli`/`ReaderT`.

If you are using `cats-effect` and you are familiar with `IOLocal`, then Context is very similar (and can be constructed on top of it).
However, `Context` allows spawning new ad-hoc `Local` instances for any effect type `F`, and not just `IO`.

You can construct an instance of `Context` in various ways, here is one using `cats-effect`.
```scala mdoc
import catcheffect._
Context.ioContext
```

### Example
When you have an instance of `Context` in scope, new `Local` instances can be spawned.
```scala mdoc
import cats._
import cats.effect._
import cats.implicits._
import cats.effect.std._

type Auth = String
def authorizedRoute[F[_]: Monad: Console](implicit L: Local[F, Auth]): F[Unit] = 
    for {
        user <- L.ask
        _ <- Console[F].println(s"doing user op with $user")
        _ <- L.local{
            L.ask.flatMap{ user =>
                Console[F].println(s"doing admin operation with $user")
            }
        }(_ => "admin")
        user <- L.ask
        _ <- Console[F].println(s"now I am working with a normal $user again")
    } yield ()

def run[F[_]: Monad: Console: Context] = 
    Context[F].use("user"){ implicit L =>
        authorizedRoute[F]
    }
```

### Other ways of constructing Context
There are several other ways to construct `Context` than to use `IO`.
If you are working in `Kleisli` a natural implementation exists.
```scala mdoc
Context.kleisli[IO]
```
Or for any instance of `Local[F, Vault]`.
```scala
implicit lazy val L: Local[IO, Vault] = ???
Context.local[IO]
```

## Catch
`Catch` responsible for throwing and catching errors.
The MTL counterpart of `Catch` is `EitherT`.
`Catch` can introduce new ad-hoc error channels that are independent of eachother.
There are various ways to construct a catch, but the simplest (given that you're working in `cats-effect`) is the following.
```scala mdoc
Catch.ioCatch
```

### Example
With an instance of `Catch` in scope, you can create locally scoped domain-specific errors.
```scala mdoc
sealed trait DomainError
case object MissingData extends DomainError
def domainFunction[F[_]: Console](implicit F: Async[F], R: Raise[F, DomainError]) = 
    for {
        xs <- F.delay((0 to 10).toList)
        _ <- R.raiseIf(MissingData)(xs.nonEmpty)
        _ <- Console[F].println("Firing the missiles!")
    } yield ()

def doDomainEffect[F[_]: Catch: Async: Console] = 
    Catch[F].use[DomainError]{ implicit R =>
        domainFunction[F]
    }.map{
        case Left(MissingData) => ???
        case Right(()) => ???
    }
```

Nested `Catch`, `Raise` and `Handle` instances are well behaved when nested and can raise errors on their completely isolated error channels.

### Other ways of constructing Catch
1. Catch can occur an instance of `Handle[F, Vault]` (or `EitherT[F, Vault, A]`)
2. Catch can occur for an instance of `Local[F, Vault]` (or `Kleisli[F, Vault, A]`) and `Concurrent[F]` via cancellation

An interesting observation is that you can in fact construct `Catch` if you have `Context` and `Concurrent`.
```scala mdoc
import org.typelevel.vault._
trait MyError
def example[F[_]: Context: Concurrent] =
    Context[F].use(Vault.empty){ implicit L => 
        Catch.fromLocal[F].use[MyError]{ implicit R => 
            Concurrent[F].unit
        }
    }
```