---
layout: post
title: "Kafka Streams DSL vs Processor API"
date: 2017-11-04
comments: true
categories: [kafka, streaming, scala]
---

Kafka Streams is a Java library for building real-time, highly scalable, fault tolerant,
distributed applications. 
The library is fully integrated with Kafka and leverages Kafka producer and consumer semantic
(e.g: partitioning, rebalancing, data retention).
What is really unique, running Kafka cluster is the only dependency required to run Kafka Stream application. 
Even local state stores are backed by Kafka topics to make the processing fault tolerant - brilliant!

Kafka Streams provides all necessary stream processing primitives like one-record-at-a-time processing, 
event time processing, windowing support and local state management. 
Application developer can choose from three different Kafka Streams APIs: DSL, Processor and KSQL.

* Kafka Streams DSL (Domain Specific Language) recommended way for most users 
because business logic can be expressed in a few lines of code.
All stateless and stateful transformations are defined using declarative, 
functional programming style (filter, map, flatMap, reduce, aggregate operations).
Kafka Stream DSL encapsulates most of the stream processing complexity
but unfortunately it also hides many useful knobs and switches. 

* Kafka Processor API provides low level, imperative way to define stream processing logic.
At first sight Processor API could look hostile but finally gives much more flexibility to developer.
This blog post shows that hand crafted stream processors might be a magnitude more efficient than
naive implementation using Kafka DSL.

* KSQL is a promise that stream processing could be expressed by anyone using SQL like language.
It offers an easy way to express stream processing transformations as an alternative to writing 
an application in a programming language such as Java.
In addition processing transformation written in SQL like language can be highly optimized 
by execution engine without any developer effort. 
Unfortunately KSQL was released recently and it is still at very early development stage.

In the first part of this blog post I'll define simple but still realistic business problem to solve.
Then you will learn how to implement this use case with Kafka Stream DSL 
and how much the processing performance is affected by this naive solution.
At this moment you could stop reading and scale-up Kafka cluster ten times to fulfill business requirements 
or you could continue reading and learn how to optimize the processing with low level Kafka Processor API.

## Business Use Case

Let's imagine large e-commerce web based platform with fabulous recommendation and advertisement subsystems.
Every client during visit gets personalized recommendations and advertisements,
the conversion is extraordinary high and platform earns additional profits from advertisers.
To build comprehensive recommendation models, 
such system needs to know everything about clients traits and their behaviour.

To make it possible, e-commerce platform reports all clients activities as unbounded stream 
of page views and events.
Every time, the client enters web page, so-called page view is sent to Kafka cluster. 
Page view defines web page attributes like request URI, referrer URI, user agent, active A/B experiments
and many more.
In addition to page view all important actions are reported as events, e.g: search, add to cart or checkout.
To get complete view of the activity stream, collected events need to be enriched by data from page views.

## Data Model

Because most of the processing logic is built with context of given client, 
page views and events are partitioned on Kafka topics by client identifier.

``` scala
type ClientId = String
case class ClientKey(clientId: ClientId)

val bob = ClientKey("bob")
val jim = ClientKey("jim")
```

Page view and event structures are different so messages are published to separate Kafka topics.
The topics key is always `ClientKey` and value is either `Pv` or `Ev` presented below.
For better examples readability page view and event payload is defined as simplified single value field.

``` scala
type PvId = String
type EvId = String

case class Pv(pvId: PvId, value: String)
case class Ev(evId: EvId, value: String, pvId: PvId) 
```

The following enriched structure `EvPv` is published to output Kafka topic using `ClientKey` as message key.
This topic is then consumed directly by advertisement and recommendation subsystem.

``` scala
case class EvPv(evId: EvId, evValue: String, pvId: Option[PvId], pvValue: Option[String])
```

## Example Scenario

For client "bob" the following messages are collected by the system.

``` scala
// initial page view
ClientKey("bob"), Pv("pv0", "/")

// a few events collected almost immediatelly
ClientKey("bob"), Ev("ev0", "show header", "pv0")
ClientKey("bob"), Ev("ev1", "show ads", "pv0")
ClientKey("bob"), Ev("ev2", "show recommendation", "pv0")

// duplicated event, welcome to distributed world
ClientKey("bob"), Pv("ev1", "show ads", "pv0")

// a dozen seconds later
ClientKey("bob"), Pv("ev3", "click recommendation", "pv0")

// out of order, early event collected before page view
ClientKey("bob"), Ev("ev0", "show header", "pv1")

// second page view
ClientKey("bob"), Pv("pv1", "/item?id=1234&reco=abc")

// event published almost immediatelly
ClientKey("bob"), Ev("ev1", "show ads", "pv1")

// a dozen minutes later, bob took coffe break before purchase
ClientKey("bob"), Ev("ev2", "add to cart", "pv1")
```

For above clickstream the following output is expected.

``` scala
ClientKey("bob"), EvPv("ev0", "show header", "pv0", "/")
ClientKey("bob"), EvPv("ev1", "show ads", "pv0", "/") // no duplicates
ClientKey("bob"), EvPv("ev2", "show recommendation", "pv0", "/")
ClientKey("bob"), EvPv("ev3", "click recommendation", "pv0", "/")

ClientKey("bob"), EvPv("ev0", "show header", None, None) // page view has not been collected yet
ClientKey("bob"), EvPv("ev1", "show ads", "pv1", "/item?id=1234&reco=abc")
ClientKey("bob"), EvPv("ev2", "add to cart", None, None) // late event out of join window :(
```

## Kafka Stream DSL

TODO

``` scala
val builder = new StreamsBuilder()

// sources
val evStream: KStream[ClientKey, Ev] = builder.stream[ClientKey, Ev](EvTopic)
val pvStream: KStream[ClientKey, Pv] = builder.stream[ClientKey, Pv](PvTopic)

// repartition events by clientKey + pvKey
val evToPvKeyMapper: KeyValueMapper[ClientKey, Ev, PvKey] =
  (clientKey, ev) => PvKey(clientKey.clientId, ev.pvId)

val evByPvKeyStream: KStream[PvKey, Ev] = evStream.selectKey(evToPvKeyMapper)

// repartition page views by clientKey + pvKey
val pvToPvKeyMapper: KeyValueMapper[ClientKey, Pv, PvKey] =
  (clientKey, pv) => PvKey(clientKey.clientId, pv.pvId)

val pvByPvKeyStream: KStream[PvKey, Pv] = pvStream.selectKey(pvToPvKeyMapper)

// join
val evPvJoiner: ValueJoiner[Ev, Pv, EvPv] = { (ev, pv) =>
  if (pv == null) {
    EvPv(ev.evId, ev.value, None, None)
  } else {
    EvPv(ev.evId, ev.value, Some(pv.pvId), Some(pv.value))
  }
}

val joinRetention = PvWindow.toMillis * 2 + 1
val joinWindow = JoinWindows.of(PvWindow.toMillis).until(joinRetention)

val evPvStream: KStream[PvKey, EvPv] = evByPvKeyStream.leftJoin(pvByPvKeyStream, evPvJoiner, joinWindow)

// repartition by clientKey + pvKey + evKey
val evPvToEvPvKeyMapper: KeyValueMapper[PvKey, EvPv, EvPvKey] =
  (pvKey, evPv) => EvPvKey(pvKey.clientId, pvKey.pvId, evPv.evId)

val evPvByEvPvKeyStream: KStream[EvPvKey, EvPv] = evPvStream.selectKey(evPvToEvPvKeyMapper)

// deduplicate
val evPvReducer: Reducer[EvPv] =
  (evPv1, _) => evPv1

val deduplicationRetention = EvPvWindow.toMillis * 2 + 1
val deduplicationWindow = TimeWindows.of(EvPvWindow.toMillis).until(deduplicationRetention)

val deduplicatedStream: KStream[Windowed[EvPvKey], EvPv] = evPvByEvPvKeyStream
  .groupByKey()
  .reduce(evPvReducer, deduplicationWindow, "evpv-store")
  .toStream()

// map key again into client id
val evPvToClientKeyMapper: KeyValueMapper[Windowed[EvPvKey], EvPv, ClientId] =
  (windowedEvPvKey, _) => windowedEvPvKey.key.clientId

val finalStream: KStream[ClientId, EvPv] = deduplicatedStream.selectKey(evPvToClientKeyMapper)

// sink
finalStream.to(EvPvTopic)

builder.build()
```

## Under the hood

TODO

## Kafka Processor API

TODO

``` scala Topology
val pvStoreName = "pv-store"
val evPvStoreName = "evpv-store"
val pvWindowProcessorName = "pv-window-processor"
val evJoinProcessorName = "ev-join-processor"
val evPvMapProcessorName = "ev-pv-processor"

val pvStore = pvStoreBuilder(pvStoreName, PvWindow)
val evPvStore = evPvStoreBuilder(evPvStoreName, EvPvWindow)

val pvWindowProcessor: ProcessorSupplier[ClientKey, Pv] =
  () => new PvWindowProcessor(pvStoreName)

val evJoinProcessor: ProcessorSupplier[ClientKey, Ev] =
  () => new EvJoinProcessor(pvStoreName, PvWindow, EvPvWindow)

val evPvMapProcessor: ProcessorSupplier[EvPvKey, EvPv] =
  () => new EvPvMapProcessor()

new Topology()
  // sources
  .addSource(PvTopic, PvTopic)
  .addSource(EvTopic, EvTopic)
  // window for page views
  .addProcessor(pvWindowProcessorName, pvWindowProcessor, PvTopic)
  // join on (clientId + pvId + evId) and deduplicate
  .addProcessor(evJoinProcessorName, evJoinProcessor, EvTopic)
  // map key again into clientId
  .addProcessor(evPvMapProcessorName, evPvMapProcessor, evJoinProcessorName)
  // sink
  .addSink(EvPvTopic, EvPvTopic, evPvMapProcessorName)
  // state stores
  .addStateStore(pvStore, pvWindowProcessorName, evJoinProcessorName)
  .addStateStore(evPvStore, evJoinProcessorName)
```

``` scala StoreBuilders
def pvStoreBuilder(storeName: String, storeWindow: FiniteDuration): StoreBuilder[WindowStore[ClientKey, Pv]] = {
  val retention = storeWindow.toMillis
  val window = storeWindow.toMillis
  val segments = 3
  val retainDuplicates = false

  Stores.windowStoreBuilder(
    Stores.persistentWindowStore(storeName, retention, segments, window, retainDuplicates),
    ClientKeySerde,
    PvSerde
  )
}

def evPvStoreBuilder(storeName: String, storeWindow: FiniteDuration): StoreBuilder[WindowStore[EvPvKey, EvPv]] = {
  val retention = storeWindow.toMillis
  val window = storeWindow.toMillis
  val segments = 3
  val retainDuplicates = false

  Stores.windowStoreBuilder(
    Stores.persistentWindowStore(storeName, retention, segments, window, retainDuplicates),
    EvPvKeySerde,
    EvPvSerde
  )
}
```

``` scala PvWindowProcessor
class PvWindowProcessor(val pvStoreName: String) extends AbstractProcessor[ClientKey, Pv] {

  private lazy val pvStore: WindowStore[ClientKey, Pv] =
    context().getStateStore(pvStoreName).asInstanceOf[WindowStore[ClientKey, Pv]]

  override def process(key: ClientKey, value: Pv): Unit = {
    context().forward(key, value)
    pvStore.put(key, value)
  }
}
```

``` scala EvJoinProcessor
class EvJoinProcessor(
  val pvStoreName: String,
  val joinWindow: FiniteDuration,
  val deduplicationWindow: FiniteDuration
) extends AbstractProcessor[ClientKey, Ev] {

  import scala.collection.JavaConverters._

  private lazy val pvStore: WindowStore[ClientKey, Pv] =
    context().getStateStore(pvStoreName).asInstanceOf[WindowStore[ClientKey, Pv]]

  private lazy val evPvStore: WindowStore[EvPvKey, EvPv] =
    context().getStateStore(pvStoreName).asInstanceOf[WindowStore[EvPvKey, EvPv]]

  override def process(key: ClientKey, ev: Ev): Unit = {
    val timestamp = context().timestamp()
    val evPvKey = EvPvKey(key.clientId, ev.pvId, ev.evId)

    if (isNotDuplicate(evPvKey, timestamp, deduplicationWindow)) {
      val pvs = storedPvs(key, timestamp, joinWindow)

      val evPvs = if (pvs.isEmpty) {
        Seq(EvPv(ev.evId, ev.value, None, None))
      } else {
        pvs
          .filter { pv =>
            pv.pvId == ev.pvId
          }
          .map { pv =>
            EvPv(ev.evId, ev.value, Some(pv.pvId), Some(pv.value))
          }
          .toSeq
      }

      evPvs.foreach { evPv =>
        context().forward(evPvKey, evPv)
        evPvStore.put(evPvKey, evPv)
      }
    }
  }

  private def isNotDuplicate(evPvKey: EvPvKey, timestamp: Long, deduplicationWindow: FiniteDuration) =
    evPvStore.fetch(evPvKey, timestamp - deduplicationWindow.toMillis, timestamp).asScala.isEmpty

  private def storedPvs(key: ClientKey, timestamp: Long, joinWindow: FiniteDuration) =
    pvStore.fetch(key, timestamp - joinWindow.toMillis, timestamp).asScala.map(_.value)
  }
```

``` scala EvPvMapProcessor
class EvPvMapProcessor extends AbstractProcessor[EvPvKey, EvPv] {
  override def process(key: EvPvKey, value: EvPv): Unit =
    context().forward(ClientKey(key.clientId), value)
}
```

## Summary

TODO