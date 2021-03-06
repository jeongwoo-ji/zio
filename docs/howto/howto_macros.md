---
id: howto_macros
title:  "Macros"
---

## Scrapping the boilerplate with macros

Many libraries come together with usage best practices and repeatable code, ZIO is no diffrent. Fortunately ZIO provides macros
to perform these repetitive tasks for you. At the moment these are only available for Scala versions `2.x`, however their equivalents
for Dotty are on our roadmap.

### Prerequisites

To enable macro expansion you need to setup your project:

- for Scala `>= 2.13` add compiler option

```scala
scalacOptions += "-Ymacro-annotations"
```

- for Scala `< 2.13` add macro paradise compiler plugin

```scala
compilerPlugin(("org.scalamacros" % "paradise"  % "2.1.1") cross CrossVersion.full)
```

## Capability accessors

### Installation

```scala
libraryDependencies += "dev.zio" %% "zio-macros" % "<zio-version>"
```

### Description

The `@accessible` macro generates _capability accessors_ into annotated module object.

```scala
import zio.{ Has, ZIO }
import zio.macros.accessible

@accessible
object AccountObserver {
  trait Service {
    def processEvent(event: AccountEvent): UIO[Unit]
  }

  // below will be autogenerated
  def processEvent(event: AccountEvent) =
    ZIO.accessM[Has[AccountObserver.Service]](_.get[Service].processEvent(event))
}
```

> **Note:** the macro can only be applied to objects which contain a trait called `Service`.


## Capability tags

### Installation

```scala
libraryDependencies += "dev.zio" %% "zio-test" % "<zio-version>"
```

### Description

The `@mockable[A]` generates _capability tags_ and _mock layer_ into annotated object.

```scala
import zio.test.mock.mockable

@mockable[AccountObserver.Service]
object AccountObserverMock
```

Will result in:

```scala
import zio.{ Has, UIO, URLayer, ZLayer }
import zio.test.mock.{ Mock, Proxy }

object AccountObserverMock extends Mock[Has[AccountObserver.Service]] {

  object ProcessEvent extends Effect[AccountEvent, Nothing, Unit]
  object RunCommand   extends Effect[Unit, Nothing, Unit]

  val compose: URLayer[Has[Proxy], AccountObserver] =
    ZLayer.fromServiceM { proxy =>
      withRuntime.map { rts =>
        new AccountObserver.Service {
          def processEvent(event: AccountEvent) = proxy(ProcessEvent, event)
          def runCommand: UIO[Unit]           = proxy(RunCommand)
        }
      }
    }
}
```
