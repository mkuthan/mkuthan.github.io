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

Because most of the processing logic is built within context of given client, 
page views and events are evenly partitioned on Kafka topics by client identifier.

``` scala
type ClientId = String
case class ClientKey(clientId: ClientId)

val bob = ClientKey("bob")
val jim = ClientKey("jim")
```

Page view and event structures are different so messages are published to separate Kafka topics.
As an event time, the collection time is used as much safer than event creation time in the client browser.
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

For client "bob" the following page views and events are collected by the system.

``` scala
// Bob enters main page
ClientKey("bob"), Pv("pv0", "/")

// A few impression events collected almost immediatelly
ClientKey("bob"), Ev("ev0", "show header", "pv0")
ClientKey("bob"), Ev("ev1", "show ads", "pv0")
ClientKey("bob"), Ev("ev2", "show recommendation", "pv0")

// There is also single duplicated event, welcome to distributed world
ClientKey("bob"), Pv("ev1", "show ads", "pv0")

// A dozen seconds later Bob clicked on one of the offers
ClientKey("bob"), Pv("ev3", "click recommendation", "pv0")

// On the offer page, early out of order event collected before page view
ClientKey("bob"), Ev("ev0", "show header", "pv1")

// Offer page view
ClientKey("bob"), Pv("pv1", "/offer?id=1234")

// An impression event published almost immediatelly
ClientKey("bob"), Ev("ev1", "show ads", "pv1")

// Bob took coffe break before purchase, a dozen minutes later ... 
ClientKey("bob"), Ev("ev2", "add to cart", "pv1")
```

For above clickstream the following output is expected.

``` scala
ClientKey("bob"), EvPv("ev0", "show header", "pv0", "/")
ClientKey("bob"), EvPv("ev1", "show ads", "pv0", "/") // no duplicates :)
ClientKey("bob"), EvPv("ev2", "show recommendation", "pv0", "/")
ClientKey("bob"), EvPv("ev3", "click recommendation", "pv0", "/")

ClientKey("bob"), EvPv("ev0", "show header", None, None) // page view has not been collected yet :(
ClientKey("bob"), EvPv("ev1", "show ads", "pv1", "/offer?id=1234")
ClientKey("bob"), EvPv("ev2", "add to cart", None, None) // late event out of join window :(
```

## Kafka Stream DSL

Now we are ready to implement stream processing using recommended Kafka Streams DSL. 
The code could be optimized but I wanted to present canonical way of using DSL.

Create two input streams for page views and events.

``` scala
val builder = new StreamsBuilder()

// sources
val evStream: KStream[ClientKey, Ev] = builder.stream[ClientKey, Ev](EvTopic)
val pvStream: KStream[ClientKey, Pv] = builder.stream[ClientKey, Pv](PvTopic)
```

Repartition topics by client and page view identifier as a prerequisite to join events with page view. 

``` scala
case class PvKey(clientId: ClientId, pvId: PvId)

val evToPvKeyMapper: KeyValueMapper[ClientKey, Ev, PvKey] =
  (clientKey, ev) => PvKey(clientKey.clientId, ev.pvId)

val evByPvKeyStream: KStream[PvKey, Ev] = evStream.selectKey(evToPvKeyMapper)

val pvToPvKeyMapper: KeyValueMapper[ClientKey, Pv, PvKey] =
  (clientKey, pv) => PvKey(clientKey.clientId, pv.pvId)

val pvByPvKeyStream: KStream[PvKey, Pv] = pvStream.selectKey(pvToPvKeyMapper)
```

Join event with page view streams by `PvKey`, 
left join is used because we are interested in events even without matched page view.
The join window duration is defined to 10 minutes.
It means, that Kafka Streams will look for messages in this and other side of the join 
10 minutes in the past and 10 minutes in the future (event time, not wall-clock time).
We are also not interested in late events out of defined window, 
so the retention is defined 2 times longer than window.
If you are interested why 1 millis needs to be added to the retention, 
ask Kafka Streams architects not me ;)

``` scala
val evPvJoiner: ValueJoiner[Ev, Pv, EvPv] = { (ev, pv) =>
  if (pv == null) {
    EvPv(ev.evId, ev.value, None, None)
  } else {
    EvPv(ev.evId, ev.value, Some(pv.pvId), Some(pv.value))
  }
}

val joinWindowDuration = 10 minutes

val joinRetention = joinWindowDuration.toMillis * 2 + 1
val joinWindow = JoinWindows.of(joinWindowDuration.toMillis).until(joinRetention)

val evPvStream: KStream[PvKey, EvPv] = evByPvKeyStream.leftJoin(pvByPvKeyStream, evPvJoiner, joinWindow)
```

Time to fight with duplicated events, 
joined stream needs to be again repartitioned by compound key: client, page view and event identifiers.
Deduplication logic is implemented as reduce function, first observed event wins. 
Deduplication window can be much shorter than join window, 
we do not expect duplicates more than 30 seconds between each other.

``` scala
case class EvPvKey(clientId: ClientId, pvId: PvId, evId: EvId)

val evPvToEvPvKeyMapper: KeyValueMapper[PvKey, EvPv, EvPvKey] =
  (pvKey, evPv) => EvPvKey(pvKey.clientId, pvKey.pvId, evPv.evId)

val evPvByEvPvKeyStream: KStream[EvPvKey, EvPv] = evPvStream.selectKey(evPvToEvPvKeyMapper)

val evPvDeduplicator: Reducer[EvPv] =
  (evPv1, _) => evPv1

val deduplicationWindowDuration = 30 seconds

val deduplicationRetention = deduplicationWindowDuration.toMillis * 2 + 1
val deduplicationWindow = TimeWindows.of(deduplicationWindowDuration.toMillis).until(deduplicationRetention)

val deduplicatedStream: KStream[Windowed[EvPvKey], EvPv] = evPvByEvPvKeyStream
  .groupByKey()
  .reduce(evPvDeduplicator, deduplicationWindow, "evpv-store")
  .toStream()
```

In the last stage the stream needs to be again repartitioned by client id for downstream subscribers.
The join results are finally published to `EvPvTopic`.

``` scala
val evPvToClientKeyMapper: KeyValueMapper[Windowed[EvPvKey], EvPv, ClientId] =
  (windowedEvPvKey, _) => windowedEvPvKey.key.clientId

val finalStream: KStream[ClientId, EvPv] = deduplicatedStream.selectKey(evPvToClientKeyMapper)

finalStream.to(EvPvTopic)
```

## Under The Hood

Kafka Stream DSL is quite descriptive, isn't it? 
But you will shortly see how much unexpected traffic to and from Kafka cluster is generated.

I like numbers so estimate the traffic based on realistic clickstream ingestion platform:

* 1 kB - average page view size 
* 600 B - average event size
* 4k - page views / seconds 
* 20k - events / seconds

It gives 24k msgs/s and 16MB/s traffic-in total, easily handled even by small Kafka cluster.

When stream of data is repartitioned before join Kafka Streams creates additional intermediate topic
partitioned by selected key. 
To be more precise two topic in our case, one for repartitioned page views second for repartitioned events.
We need to add 24k msgs/s and 16MB/s more traffic-in to the calculation.

When streams of data are joined using window, Kafka Streams sends this and other side of the join
to intermediate topics again. Even if you don't need fault tolerance,
logging into Kafka can not be disabled using DSL.
You cannot also get rid of window for "this" side of the join (window for events).
Add 24k msgs/s and 16MB/s more traffic-in to the calculation again.

To deduplicate events, joined stream goes again into Kafka Streams intermediate topic.
Add 20k msgs/s and (1kB + 1.6kB) * 20k = 52MB/s more traffic-in to the calculation again.

The last repartitioning by client identifier adds 20k msgs/s and 52MB/s more traffic-in.

Finally, instead **24k** msgs/s and **16MB/s** traffic-in we have got
**112k** messages per second and **152MB** traffic-in.
And I did not event count traffic from internal topics replication and standby replicas 
[recommended for resiliency](https://docs.confluent.io/current/streams/developer-guide.html#recommended-configuration-parameters-for-resiliency). 

And this calculation is only for simple join of events and pages views generated by 
local e-commerce platform in Central Europe country. 
I could easily imagine much more complex stream processing, with tens of reparations, joins and aggregations. 

## Kafka Processor API

Now it's time to check Processor API and figure out how to optimize our stream processing.

Create the sources.

``` scala
new Topology()
  .addSource("ev-source", EvTopic)
  .addSource("pv-source", PvTopic)
```

Because we need to join incoming event with collected page view in the past, 
create processor which stores page view in windowed store. 
Events do not need to be remembered in window store and all.
The processor put observed page views into window store for joining in the next processor.

``` scala
class PvWindowProcessor(val pvStoreName: String) extends AbstractProcessor[ClientKey, Pv] {

  private lazy val pvStore: WindowStore[ClientKey, Pv] =
    context().getStateStore(pvStoreName).asInstanceOf[WindowStore[ClientKey, Pv]]

  override def process(key: ClientKey, value: Pv): Unit = {
    context().forward(key, value)
    pvStore.put(key, value)
  }
}
```

Store for page views are configured with the same size of window and retention. 
This store also removes duplicated page views for free (retainDuplicates parameter).
Because join window is typically quite long (minutes) the store is configured as fault tolerant (logging enabled).
Even if one of the stream instances fails, 
another one will continue processing with persistent window state built by failed node, cool!
Finally, the internal kafka topic can be easily configured using loggingConfig map 
(replication factor, number of partitions, etc.).

``` scala 
val retention = storeWindow.toMillis
val window = storeWindow.toMillis
val segments = 3
val retainDuplicates = false

val loggingConfig = Map[String, String]()

Stores.windowStoreBuilder(
  Stores.persistentWindowStore(storeName, retention, segments, window, retainDuplicates),
  ClientKeySerde,
  PvSerde
).withLoggingEnabled(loggingConfig.asJava)
```

Add page view processor to the topology and connect with page view source upstream.

``` scala
val pvWindowProcessor: ProcessorSupplier[ClientKey, Pv] =
  () => new PvWindowProcessor("pv-window-store")
  
new Topology()
  (...)
  .addProcessor("pv-window-processor, pvWindowProcessor, "pv-source")
```

Now, it's time for event and page view join processor, hearth of the processing topology.
I't seems to be complex but this processor also deduplicates joined stream using `evPvStore`.

First processor performs a lookup for previous joined `PvEv`, 
if found skip the processing because `EvPv` has been already processed.

Next, if page view does not exist for given event (`pvs.isEmpty`) returns event itself (left join semantics).
If there are matched page views from given client, try to match event using simple filter `pv.pvId == ev.pvId`.
We don't need perform any repartitioning to do that, only gets all page views from given client
and join in the processor itself.

Perceptive reader noticed that processor also change the key from `ClientId` to `EvPvKey` 
for deduplication purposes. Without any repartitioning, due to the fact that new key 
is more detailed that original one. 

``` scala
class EvJoinProcessor(
  val pvStoreName: String,
  val evPvStoreName: String,
  val joinWindow: FiniteDuration,
  val deduplicationWindow: FiniteDuration
) extends AbstractProcessor[ClientKey, Ev] {

  import scala.collection.JavaConverters._

  private lazy val pvStore: WindowStore[ClientKey, Pv] =
    context().getStateStore(pvStoreName).asInstanceOf[WindowStore[ClientKey, Pv]]

  private lazy val evPvStore: WindowStore[EvPvKey, EvPv] =
    context().getStateStore(evPvStoreName).asInstanceOf[WindowStore[EvPvKey, EvPv]]

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

Again store for deduplication needs to be configured.
Because deduplication is done in very short window (10 seconds or so), the logging is disable at all.
If one of the stream instance fails, we will get some duplicates during this short window, not a big deal.

``` scala StoreBuilders
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

Add join processor to the topology and connect with event source upstream.

``` scala
val evJoinProcessor: ProcessorSupplier[ClientKey, Ev] =
  () => new EvJoinProcessor("ev-join-processor", "pv-window-store", "ev-pv-store", JoinWindow, DedupliationWindow)

new Topology()
  (...)
  .addProcessor("ev-join-processor", evJoinProcessor, "ev-source")
```

The last processor maps compound key `EvPvKey` again into `ClientId`. 
Because client identifier is part of the compound key, repartitioning is not needed.

``` scala
class EvPvMapProcessor extends AbstractProcessor[EvPvKey, EvPv] {
  override def process(key: EvPvKey, value: EvPv): Unit =
    context().forward(ClientKey(key.clientId), value)
}
```

Add the map processor to the topology.

``` scala
val evPvMapProcessor: ProcessorSupplier[EvPvKey, EvPv] =
  () => new EvPvMapProcessor()

new Topology()
  (...)
  .addProcessor("ev-pv-map-processor", evPvMapProcessor, "ev-pv-join-processor")
```

Publish join result to Kafka topic, register configured stores and connect them to the processors.

``` scala
new Topology()
  (...)
  .addSink("ev-pv-sink", EvPvTopic, "ev-pv-map-processor")
  // stores
  .addStateStore(pvStore, "pv-window-processor", "ev-pv-join-processor")
  .addStateStore(evPvStore, "ev-pv-join-processor")
```

Full topology definition is presented below.

``` scala
val pvStoreName = "pv-store"
val evPvStoreName = "ev-pv-store"
val pvWindowProcessorName = "pv-window-processor"
val evJoinProcessorName = "ev-join-processor"
val evPvMapProcessorName = "ev-pv-processor"

val pvStore = pvStoreBuilder(pvStoreName, PvWindow)
val evPvStore = evPvStoreBuilder(evPvStoreName, EvPvWindow)

val pvWindowProcessor: ProcessorSupplier[ClientKey, Pv] =
  () => new PvWindowProcessor(pvStoreName)

val evJoinProcessor: ProcessorSupplier[ClientKey, Ev] =
  () => new EvJoinProcessor(pvStoreName, evPvStoreName, PvWindow, EvPvWindow)

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

Let's count Kafka Streams internal topics overhead for Processor API version.
Wait, there is only one topic, for page view join window!

**4k** messages per second and **4MB** traffic-in overhead, not more. 

## Summary

Dear readers, are you still we me? 
Let's digest and summarize this long post with not so easy code.

* Kafka DSL looks great at first, functional and declarative API sells the product, no doubts.
* Unfortunately Kafka DSL hides a lot of internals which should be exposed via the API 
(stores configuration, join semantics, repartitioning).
* Processor API seems to be more complex and less sexy than DSL.
* But Processor API allows you to create hand-crafted, very efficient stream processing logic.
* I did not present any Kafka Streams test (what's the shame - sorry) 
but I think that testing would be easier with Processor API than DSL. 
With DSL it has to be an integration test, processors can be easily unit tested in separation.
* As Scala developer I prefer Processor API than DSL, 
e.g. Scala compiler could not infer KStream generic types. 
It's a pleasure to work with processor and fluent Topology APIs. 
* I'm really keen on KSQL future,
it would be great to get optimized engine like Spark Catalyst eventually.

As always working code is published on 
[github.com/mkuthan/example-kafkastreams](https://github.com/mkuthan/example-kafkastreams).
The project is configured with [Embedded Kafka](https://github.com/manub/scalatest-embedded-kafka)
and does not require any additional setup. 
Just uncomment either DSL or Processor API version and run main class.

Enjoy!
