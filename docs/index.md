---
id: index 
title: "Getting Started with ZIO Kafka"
sidebar_label: "Getting Started"
---

[ZIO Kafka](https://github.com/zio/zio-kafka) is a Kafka client for ZIO. It provides a purely functional, streams-based interface to the Kafka
client and integrates effortlessly with ZIO and ZIO Streams. Often zio-kafka programs have a _higher_ throughput than
programs that use the Java Kafka client directly (see section [Performance](#performance) below).

@PROJECT_BADGES@ [![Scala Steward badge](https://img.shields.io/badge/Scala_Steward-helping-blue.svg?style=flat&logo=data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAA4AAAAQCAMAAAARSr4IAAAAVFBMVEUAAACHjojlOy5NWlrKzcYRKjGFjIbp293YycuLa3pYY2LSqql4f3pCUFTgSjNodYRmcXUsPD/NTTbjRS+2jomhgnzNc223cGvZS0HaSD0XLjbaSjElhIr+AAAAAXRSTlMAQObYZgAAAHlJREFUCNdNyosOwyAIhWHAQS1Vt7a77/3fcxxdmv0xwmckutAR1nkm4ggbyEcg/wWmlGLDAA3oL50xi6fk5ffZ3E2E3QfZDCcCN2YtbEWZt+Drc6u6rlqv7Uk0LdKqqr5rk2UCRXOk0vmQKGfc94nOJyQjouF9H/wCc9gECEYfONoAAAAASUVORK5CYII=)](https://scala-steward.org)

## Introduction

Apache Kafka is a distributed event streaming platform that acts as a distributed publish-subscribe messaging system. It enables us to build distributed streaming data pipelines and event-driven applications.

Kafka has a mature Java client for producing and consuming events, but it has a low-level API. ZIO Kafka is a ZIO native client for Apache Kafka. It has a high-level streaming API on top of the Java client. So we can produce and consume events using the declarative concurrency model of ZIO Streams.

## Installation

In order to use this library, we need to add the following line in our `build.sbt` file:

```scala
libraryDependencies += "dev.zio" %% "zio-kafka"         % "@VERSION@"
libraryDependencies += "dev.zio" %% "zio-kafka-testkit" % "@VERSION@" % Test
```

Snapshots are available on Sonatype's snapshot repository https://oss.sonatype.org/content/repositories/snapshots.
[Browse here](https://oss.sonatype.org/content/repositories/snapshots/dev/zio/zio-kafka_3/) to find available versions.

For `zio-kafka-testkit` together with Scala 3, you also need to add the following to your `build.sbt` file:

```scala
excludeDependencies += "org.scala-lang.modules" % "scala-collection-compat_2.13"
```

## Example

Let's write a simple Kafka producer and consumer using ZIO Kafka with ZIO Streams. Before everything, we need a running instance of Kafka. We can do that by saving the following docker-compose script in the `docker-compose.yml` file and run `docker-compose up`:

```yaml
version: '2'
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    ports:
      - 22181:2181
  
  kafka:
    image: confluentinc/cp-kafka:latest
    depends_on:
      - zookeeper
    ports:
      - 29092:29092
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092,PLAINTEXT_HOST://localhost:29092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
```

Now, we can run our ZIO Kafka Streaming application:

```scala
import zio._
import zio.kafka.consumer._
import zio.kafka.producer.{Producer, ProducerSettings}
import zio.kafka.serde._
import zio.stream.ZStream

object MainApp extends ZIOAppDefault {
  val producer: ZStream[Producer, Throwable, Nothing] =
    ZStream
      .repeatZIO(Random.nextIntBetween(0, Int.MaxValue))
      .schedule(Schedule.fixed(2.seconds))
      .mapZIO { random =>
        Producer.produce[Any, Long, String](
          topic = "random",
          key = random % 4,
          value = random.toString,
          keySerializer = Serde.long,
          valueSerializer = Serde.string
        )
      }
      .drain

  val consumer: ZStream[Consumer, Throwable, Nothing] =
    Consumer
      .plainStream(Subscription.topics("random"), Serde.long, Serde.string)
      .tap(r => Console.printLine(r.value))
      .map(_.offset)
      .aggregateAsync(Consumer.offsetBatches)
      .mapZIO(_.commit)
      .drain

  def producerLayer =
    ZLayer.scoped(
      Producer.make(
        settings = ProducerSettings(List("localhost:29092"))
      )
    )

  def consumerLayer =
    ZLayer.scoped(
      Consumer.make(
        ConsumerSettings(List("localhost:29092")).withGroupId("group")
      )
    )

  override def run =
    producer.merge(consumer)
      .runDrain
      .provide(producerLayer, consumerLayer)
}
```

## Resources

- [ZIO Kafka tutorial](https://zio.dev/guides/tutorials/producing-consuming-data-from-kafka-topics/)
- [Making ZIO-Kafka Safer and Faster in 2023](https://www.youtube.com/watch?v=MJoRwEyyVxM) by Erik van Oosten (November 2023)
- [ZIO Kafka with Scala: A Tutorial](https://www.youtube.com/watch?v=ExFjjczwwHs) by Rock the JVM (August 2021)
- [Streaming microservices with ZIO and Kafka](https://scalac.io/streaming-microservices-with-zio-and-kafka/) by Aleksandar Skrbic (February 2021)
- [ZIO WORLD - ZIO Kafka](https://www.youtube.com/watch?v=GECv1ONieLw) by Aleksandar Skrbic (March 2020) — Aleksandar Skrbic presented ZIO Kafka, a critical library for the modern Scala developer, which hides some of the complexities of Kafka.

## Adopters

Here is a partial list of companies using zio-kafka in production.

Want to see your company here? [Submit a PR](https://github.com/zio/zio-kafka/edit/master/docs/index.md)!

* [Conduktor](https://www.conduktor.io)
* [KelkooGroup](https://www.kelkoogroup.com)
* [Rocker](https://rocker.com)

## Performance

By default, zio-kafka programs process partitions in parallel. The default java-kafka client does not provide parallel
processing. Of course, there is some overhead in buffering records and distributing them to the fibers that need them.
On 2024-11-23, we estimated that zio-kafka consumes faster than the java-kafka client when processing takes more than
~1.2ms per 1000 records. The precise time depends on many factors. Please
see [this article](https://day-to-day-stuff.blogspot.com/2024/12/zio-kafka-faster-than-java-kafka.html) for more
details.

If you do not care for the convenient ZStream based API that zio-kafka brings, and latency is of absolute importance,
using the java based Kafka client directly is still the better choice.
