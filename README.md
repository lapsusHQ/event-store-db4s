# Dolphin<a href="https://typelevel.org/cats/"><img src="https://typelevel.org/cats/img/cats-badge.svg" height="40px" align="right" alt="Cats friendly" />

[![Continuous Integration](https://github.com/lapsusHQ/dolphin/actions/workflows/ci.yml/badge.svg?branch=main)](https://github.com/lapsusHQ/dolphin/actions/workflows/ci.yml)
[![Clean](https://github.com/lapsusHQ/dolphin/actions/workflows/clean.yml/badge.svg)](https://github.com/lapsusHQ/dolphin/actions/workflows/clean.yml)
[![GitHub issues](https://img.shields.io/github/issues/lapsusHQ/dolphin)](https://github.com/lapsusHQ/dolphin/issues)
[![Version](https://img.shields.io/badge/version-0.0--`Latest%20Commit%20Hash`--SNAPSHOT-blue)](https://github.com/lapsusHQ/dolphin)

## Table of Contents

* [Table of Contents](#table-of-contents)
* [Introduction](#introduction)
* [Disclaimer](#disclaimer)
* [Installation](#installation)
* [Usage](#usage)
  * [Append to a stream](#append-to-a-stream)
  * [Read from a stream](#read-from-a-stream)
  * [Subscribe to a stream](#subscribe-to-a-stream)
  * [Projections](#projections)
* [Roadmap](#roadmap)
* [Note](#note)

## Introduction

EventStoreDB is an open-source state-transition database, designed for businesses that are ready to harness the true
power of event-driven architecture. It is a purpose-built database for event-driven applications, with a focus on high
performance, scalability, and reliability.

## Disclaimer

⚠️ Dolphin is a Scala wrapper for the Java client of EventStoreDB. It is a work in progress and is not ready nor
recommended for production use.

## Installation

Add the following to your `build.sbt` file:

```scala
libraryDependencies ++= Seq("io.github.lapsushq" %% "dolphin-core" % "0.0-`Latest Commit Hash`-SNAPSHOT", "io.github.lapsushq" %% "dolphin-circe" % "0.0-`Latest Commit Hash`-SNAPSHOT")
```

## Usage

EventStoreDB distinguishes between a normal session and a persistent session. A normal session is a volatile session,
which means that reads operate on the disk without the possibility of acknowledging. A persistent session, in turn,
is a session that reads from the disk and provides a mechanism to acknowledge the read, in turn, you can not write with
this type of subscription. This means that a persistent
session could perform slower than the normal session.

### Append to a stream

```scala
import dolphin.*
import cats.effect.{IO, IOApp}
import org.typelevel.log4cats.SelfAwareStructuredLogger
import org.typelevel.log4cats.slf4j.Slf4jLogger
import fs2.Stream

object Main extends IOApp.Simple {

  implicit val logger: SelfAwareStructuredLogger[IO] = Slf4jLogger.getLogger[IO]

  override def run: IO[Unit] =
    (for {
      session <- VolatileSession.stream[IO](Config.Default)
      _ <- Stream.eval(
        session.appendToStream(
          "ShoppingCart",
          """{"id": "9b188885-04a8-4ae0-b8a4-74a82c17d2ec", "value": 1}""".getBytes,
          Array.emptyByteArray,
          "Counter"
        )
      )
    } yield ())
      .compile
      .drain
}
```

### Read from a stream

```scala
import dolphin.*
import dolphin.setting.ReadFromStreamSettings

import cats.effect.{IO, IOApp}

import org.typelevel.log4cats.SelfAwareStructuredLogger
import org.typelevel.log4cats.slf4j.Slf4jLogger
import fs2.Stream


object Main extends IOApp.Simple {

  implicit val logger: SelfAwareStructuredLogger[IO] = Slf4jLogger.getLogger[IO]

  override def run: IO[Unit] =
    (for {
      session <- VolatileSession.stream[IO](Config.Default)
      read <- Stream.eval(session.readStream("ShoppingCart", ReadFromStreamSettings.Default))
      data <- read.getEventData
      _ <- Stream.eval(IO.println(new String(data))) // {"id": "9b188885-04a8-4ae0-b8a4-74a82c17d2ec", "value": 1}
    } yield ())
      .compile
      .drain
}
```

EventStoreDB provides a mechanism to subscribe to a stream. This means that the client can subscribe to a stream and
receive all the events that are appended to the stream. The client can also acknowledge the events that are received (if
created with a persistent session).

### Subscribe to a stream

There are two ways to subscribe to a stream. The first way is to use the `subscribeToStream` method on the session. This
will return a `Stream` of `Message` objects. The second way is to use the `subscribeToStream` method on the session and
provide a `MessageHandler`. This will return a `Resource` of `Unit`.

- With `subscribeToStream` of `Stream`:

```scala
import dolphin.*
import cats.effect.{IO, IOApp}
import org.typelevel.log4cats.SelfAwareStructuredLogger
import org.typelevel.log4cats.slf4j.Slf4jLogger
import fs2.Stream

import java.util.UUID
import scala.concurrent.duration.*

object Main extends IOApp.Simple {

  implicit val logger: SelfAwareStructuredLogger[IO] = Slf4jLogger.getLogger[IO]

  private def program: Stream[IO, Unit] =
    for {
      session <- VolatileSession.stream[IO](Config.Default)
      _ <- Stream
        .iterateEval(UUID.randomUUID())(_ => IO(UUID.randomUUID()))
        .evalMap { uuid =>
          session
            .appendToStream(
              "ShoppingCart",
              s"""{"id": "${uuid}", "value": 1}""".getBytes,
              Array.emptyByteArray,
              "Counter"
            )
        }
        .metered(10.seconds)
        .concurrently {
          session.subscribeToStream("ShoppingCart").evalMap {
            case Message.Event(_, event, _) => logger.info(new String(event.getEventData))
            case Message.Error(_, error) => logger.error(s"Received error: ${error}")
            case Message.Cancelled(_) => logger.info("Received cancellation")
            case Message.Confirmation(_) => logger.info("Received confirmation")
          }
        }
    } yield ()

  override def run: IO[Unit] = program.compile.drain

}
```

- With `subscribeToStream` of `MessageHandler` (i.e. `Message[F, VolatileConsumer[F]] => F[Unit]`):

```scala
import dolphin.*

import cats.effect.{IO, IOApp}
import org.typelevel.log4cats.SelfAwareStructuredLogger
import org.typelevel.log4cats.slf4j.Slf4jLogger

object Main extends IOApp.Simple {

  implicit val logger: SelfAwareStructuredLogger[IO] = Slf4jLogger.getLogger[IO]

  private val handlers: Message[IO, VolatileConsumer[IO]] => IO[Unit] = {
    case Message.Event(consumer, event, retryCount) =>
      logger.info(s"Received event: $event")
    case Message.Error(consumer, error) =>
      logger.error(error)(s"Received error: $error")
    case Message.Cancelled(consumer) =>
      logger.info(s"Received cancellation")
    case Message.Confirmation(consumer) =>
      logger.info(s"Received confirmation")
  }

  override def run: IO[Unit] =
    (for {
      session <- VolatileSession.resource[IO](Config.Default)
      _ <- session.subscribeToStream("ShoppingCart", handlers)

    } yield ()).useForever

}
```

## Roadmap

## Note

- This project is not affiliated with EventStoreDB. For further information about EventStoreDB, please
  visit [EventStoreDB](https://eventstore.com/).
- For further information about the Java client, please
  visit [EventStoreDB Java Client](https://github.com/EventStore/EventStoreDB-Client-Java).
- There's a lot to change/improve, please feel free to open an issue if you have any questions or suggestions, or if you
  find any bugs.
- For further information on usage and examples, please visit [Dolphin Integration Tests](modules/tests/src/it/scala/).


