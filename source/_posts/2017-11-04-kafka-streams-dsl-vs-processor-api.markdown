---
layout: post
title: "Kafka Streams DSL vs Processor API"
date: 2017-11-04
comments: true
categories: [kafka, streaming, scala]
---

[Kafka Streams](https://docs.confluent.io/current/streams/index.html) is a Java library 
for building real-time, highly scalable, fault tolerant, distributed applications. 
The library is fully integrated with [Kafka](https://kafka.apache.org/documentation/) and leverages 
Kafka producer and consumer semantics (e.g: partitioning, rebalancing, data retention and compaction).
What is really unique, a running Kafka cluster is the only dependency required to run Kafka Stream application.
Even local state stores are backed by Kafka topics to make the processing fault tolerant - brilliant!

Kafka Streams provides all necessary stream processing primitives like one-record-at-a-time processing, 
event time processing, windowing support and local state management. 
Application developer can choose from three different Kafka Streams APIs: DSL, Processor or KSQL.

* [Kafka Streams DSL](https://docs.confluent.io/current/streams/developer-guide.html#kafka-streams-dsl) 
(Domain Specific Language) recommended way for most users 
because business logic can be expressed in a few lines of code.
All stateless and stateful transformations are defined using declarative, 
functional programming style (filter, map, flatMap, reduce, aggregate operations).
Kafka Stream DSL encapsulates most of the stream processing complexity
but unfortunately it also hides many useful knobs and switches. 

* [Kafka Processor API](https://docs.confluent.io/current/streams/developer-guide.html#processor-api) 
provides a low level, imperative way to define stream processing logic.
At first sight Processor API could look hostile but finally gives much more flexibility to developer.
This blog post shows that hand-crafted stream processors might be a magnitude more efficient than
a naive implementation using Kafka DSL.

* [KSQL](https://www.confluent.io/product/ksql/) 
is a promise that stream processing could be expressed by anyone using SQL as the language.
It offers an easy way to express stream processing transformations as an alternative to writing 
an application in a programming language such as Java.
Moreover, processing transformation written in SQL like language can be highly optimized
by execution engine without any developer effort. 
KSQL was released recently and it is still at very early development stage.

In the first part of this blog post I'll define simple but still realistic business problem to solve.
Then you will learn how to implement this use case with Kafka Stream DSL 
and how much the processing performance is affected by this naive solution.
At this moment you could stop reading and scale-up Kafka cluster ten times to fulfill business requirements 
or you could continue reading and learn how to optimize the processing with low level Kafka Processor API.

## Business Use Case

Let's imagine web based e-commerce platform with fabulous recommendation and advertisement subsystems.
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

Page view and event structures are different so messages are published to separate Kafka topics
using collection (receive) time as event time. 
Our system should not rely on page view or event creation time due to high client clocks variance.
The topic key is always `ClientKey` and value is either `Pv` or `Ev` presented below.
For better examples readability page view and event payload is defined as simplified single value field.
Events are uniquely identified by `pvId` and `evId` pair, `pvId` could be a random identifier, `evId` 
a sequence number.

``` scala
type PvId = String
type EvId = String

case class Pv(pvId: PvId, value: String)
case class Ev(evId: EvId, value: String, pvId: PvId) 
```

Enriched results `EvPv` is published to output Kafka topic using `ClientKey` as message key.
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

// On the offer page, out of order, early event collected before page view
ClientKey("bob"), Ev("ev0", "show header", "pv1")

// Offer page view
ClientKey("bob"), Pv("pv1", "/offer?id=1234")

// An impression event published almost immediatelly
ClientKey("bob"), Ev("ev1", "show ads", "pv1")

// A dozen minutes later (bob took coffe break before purchase) 
ClientKey("bob"), Ev("ev2", "add to cart", "pv1")
```

For above clickstream the following enriched events output stream is expected.

``` scala
// Events from main page without duplicates
ClientKey("bob"), EvPv("ev0", "show header", "pv0", "/")
ClientKey("bob"), EvPv("ev1", "show ads", "pv0", "/")
ClientKey("bob"), EvPv("ev2", "show recommendation", "pv0", "/")
ClientKey("bob"), EvPv("ev3", "click recommendation", "pv0", "/")

// Events from offer page, somehow incomplete due to streaming semantics limitations
ClientKey("bob"), EvPv("ev0", "show header", None, None)
ClientKey("bob"), EvPv("ev1", "show ads", "pv1", "/offer?id=1234")
ClientKey("bob"), EvPv("ev2", "add to cart", None, None)
```

## Kafka Stream DSL

Now we are ready to implement above use case with recommended Kafka Streams DSL. 
The code could be optimized but I would like present canonical way of using DSL
without thinking about DSL internals.

Create two input streams for page views and events 
connected to "clickstream.events" and "clickstream.page_views" Kafka topics.

``` scala
val builder = new StreamsBuilder()

val evStream = builder.stream[ClientKey, Ev]("clickstream.events")
val pvStream = builder.stream[ClientKey, Pv]("clickstream.page_views")
```

Repartition topics by client and page view identifiers `PvKey` 
as a prerequisite to join events with page view. 

``` scala
case class PvKey(clientId: ClientId, pvId: PvId)

val evToPvKeyMapper: KeyValueMapper[ClientKey, Ev, PvKey] =
  (clientKey, ev) => PvKey(clientKey.clientId, ev.pvId)

val evByPvKeyStream = evStream.selectKey(evToPvKeyMapper)

val pvToPvKeyMapper: KeyValueMapper[ClientKey, Pv, PvKey] =
  (clientKey, pv) => PvKey(clientKey.clientId, pv.pvId)

val pvByPvKeyStream = pvStream.selectKey(pvToPvKeyMapper)
```

Join event with page view streams by selected `PvKey`, 
left join is used because we are interested also in events without matched page view.
The join window duration is set to 10 minutes.
It means, that Kafka Streams will look for messages in "this" and "other" side of the join 
10 minutes in the past and 10 minutes in the future (using event time, not wall-clock time).
Because we are also not interested in late events out of defined window, 
the retention is defined 2 times longer than window.
If you are interested why 1 millis needs to be added to the retention, 
please ask Kafka Streams architects not me ;)

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

val evPvStream = evByPvKeyStream.leftJoin(pvByPvKeyStream, evPvJoiner, joinWindow)
```

Now it's time to fight with duplicated enriched events, 
joined stream needs to be repartitioned again by compound key of client, page view and event identifiers `EvPvKey`.
Deduplication logic is implemented as lightweight reduce function, first observed event wins. 
Deduplication window can be much shorter than join window, 
we do not expect duplicates more than 10 seconds between each other.

``` scala
case class EvPvKey(clientId: ClientId, pvId: PvId, evId: EvId)

val evPvToEvPvKeyMapper: KeyValueMapper[PvKey, EvPv, EvPvKey] =
  (pvKey, evPv) => EvPvKey(pvKey.clientId, pvKey.pvId, evPv.evId)

val evPvByEvPvKeyStream = evPvStream.selectKey(evPvToEvPvKeyMapper)

val evPvDeduplicator: Reducer[EvPv] =
  (evPv1, _) => evPv1

val deduplicationWindowDuration = 10 seconds

val deduplicationRetention = deduplicationWindowDuration.toMillis * 2 + 1
val deduplicationWindow = TimeWindows.of(deduplicationWindowDuration.toMillis).until(deduplicationRetention)

val deduplicatedStream = evPvByEvPvKeyStream
  .groupByKey()
  .reduce(evPvDeduplicator, deduplicationWindow, "evpv-store")
  .toStream()
```

In the last stage the stream needs to be repartitioned again by client id 
and published to "clickstream.events_enriched" Kafka topic for downstream subscribers.
In the same step mapper gets rid of windowed key produced by windowed reduce function.

``` scala
val evPvToClientKeyMapper: KeyValueMapper[Windowed[EvPvKey], EvPv, ClientId] =
  (windowedEvPvKey, _) => windowedEvPvKey.key.clientId

val finalStream = deduplicatedStream.selectKey(evPvToClientKeyMapper)

finalStream.to("clickstream.events_enriched")
```


## Under The Hood

Kafka Stream DSL is quite descriptive, isn't it? 
Especially developers with strong functional programming skills appreciate overall design.
But you will shortly see how much unexpected traffic to Kafka cluster is generated during runtime.

I like numbers so estimate the traffic, based on real clickstream ingestion platform I develop on daily basis:

* 1 kB - average page view size 
* 600 B - average event size
* 4k - page views / second 
* 20k - events / second

It gives 24k msgs/s and 16MB/s traffic-in total, the traffic easily handled even by small Kafka cluster.

When stream of data is repartitioned Kafka Streams creates additional intermediate topic
and publish on the topic whole traffic partitioned by selected key. 
To be more precise it happens twice in our case, for repartitioned page views and events before join.
We need to add 24k msgs/s and 16MB/s more traffic-in to the calculation.

When streams of data are joined using window, Kafka Streams sends both sides of the join
to two intermediate topics again. Even if you don't need fault tolerance,
logging into Kafka cannot be disabled using DSL.
You cannot also get rid of window for "this" side of the join (window for events), more about it later on. 
Add 24k msgs/s and 16MB/s more traffic-in to the calculation again.

To deduplicate events, joined stream goes again into Kafka Streams intermediate topic.
Add 20k msgs/s and (1kB + 1.6kB) * 20k = 52MB/s more traffic-in to the calculation again.

The last repartitioning by client identifier adds 20k msgs/s and 52MB/s more traffic-in.

Finally, instead **24k** msgs/s and **16MB/s** traffic-in we have got
**112k** msgs/s and **152MB** traffic-in.
And I did not even count traffic from internal topics replication and standby replicas 
[recommended for resiliency].(https://docs.confluent.io/current/streams/developer-guide.html#recommended-configuration-parameters-for-resiliency). 

Be aware that this is calculation for simple join of events and pages views generated by 
local e-commerce platform in central Europe country (~20M clients). 
I could also easily imagine much more complex stream topology, with tens of reparations, joins and aggregations.

If you are not careful your Kafka Streams application could easily kill your Kafka cluster.
At least our application did it once. Application deployed on 10 [Mesos](http://mesos.apache.org/) 
nodes (4CPU, 4GB RAM) almost killed Kafka cluster deployed also on 10 physical machines (32CPU, 64GB RAM, SSD)
Application was started after some time of inactivity and processed 3 hours of retention in 5 minutes
(yep, it's a well known vulnerability until 
[KIP-13](https://cwiki.apache.org/confluence/display/KAFKA/KIP-13+-+Quotas) is open).

## Kafka Processor API

Now it's time to check Processor API and figure out how to optimize our stream topology.

Create the sources from input topics.

``` scala
new Topology()
  .addSource("ev-source", "clickstream.events")
  .addSource("pv-source", "clickstream.page_views")
```

Because we need to join an incoming event with the collected page view in the past,
create processor which stores page view in windowed store. 
The processor puts observed page views into window store for joining in the next processor.
The processed page views do not even need to be forwarded to downstream, this is a terminal processor for this stream.

``` scala
class PvWindowProcessor(val pvStoreName: String) extends AbstractProcessor[ClientKey, Pv] {

  private lazy val pvStore: WindowStore[ClientKey, Pv] =
    context().getStateStore(pvStoreName).asInstanceOf[WindowStore[ClientKey, Pv]]

  override def process(key: ClientKey, value: Pv): Unit =
    pvStore.put(key, value)
}
```

Store for page views is configured with the same size of window and retention.
This store also removes duplicated page views for free (retainDuplicates parameter).
Because join window is typically quite long (minutes) the store should be fault tolerant (logging enabled).
Even if one of the stream instances fails, 
another one will continue processing with persistent window state built by failed node, cool!
Finally, the internal kafka topic can be easily configured using loggingConfig map 
(replication factor, number of partitions, etc.).

``` scala
val pvStoreWindowDuration = 10 minutes
 
val retention = pvStoreWindowDuration.toMillis
val window = pvStoreWindowDuration.toMillis
val segments = 3
val retainDuplicates = false

val loggingConfig = Map[String, String]()

val pvWindowStore = Stores.windowStoreBuilder(
  Stores.persistentWindowStore("pv-window-store", retention, segments, window, retainDuplicates),
  ClientKeySerde,
  PvSerde
).withLoggingEnabled(loggingConfig)
```

The first optimization you could observe is that in our scenario only one window store is created - for page views.
The window store for events is not needed, if page view is collected by system after event it does not trigger new join.

Add page view processor to the topology and connect with page view source upstream.

``` scala
val pvWindowProcessor: ProcessorSupplier[ClientKey, Pv] =
  () => new PvWindowProcessor("pv-window-store")
  
new Topology()
  (...)
  .addProcessor("pv-window-processor", pvWindowProcessor, "pv-source")
```

Now, it's time for event and page view join processor, heart of the topology.
It seems to be complex but this processor also deduplicates joined stream using `evPvStore`.

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

First processor performs a lookup for previous joined `PvEv` by `PvEvKey`.
If `PvEv` is found the processing is skip because `EvPv` has been already processed.

Next, if page view does not exist for given event (`pvs.isEmpty`) processor returns event itself (left join semantics).

If there are matched page views from given client, 
try to match page view to event using simple filter `pv.pvId == ev.pvId`.
We don't need any repartitioning to do that, only gets all page views from given client 
and join with event in the processor itself. 
It should be very efficient because every client generates up do hundred page views in 10 minutes.

Perceptive reader noticed that processor also changes the key from `ClientId` to `EvPvKey`
for deduplication purposes. Without any repartitioning, due to the fact that new key 
is more detailed that original one. Everything is still within given client context. 

As before windowed store for deduplication needs to be configured.
Because deduplication is done in a very short window (10 seconds or so),
the logging to backed internal Kafka topic is disable at all.
If one of the stream instance fails, we will get some duplicates during this short window, not a big deal.

``` scala StoreBuilders
val evPvStoreWindowDuration = 10 seconds

val retention = evPvStoreWindowDuration.toMillis
val window = evPvStoreWindowDuration.toMillis
val segments = 3
val retainDuplicates = false

val evPvStore = Stores.windowStoreBuilder(
  Stores.persistentWindowStore("ev-pv-window-store", retention, segments, window, retainDuplicates),
  EvPvKeySerde,
  EvPvSerde
)
```

Add join processor to the topology and connect with event source upstream.

``` scala
val evJoinProcessor: ProcessorSupplier[ClientKey, Ev] =
  () => new EvJoinProcessor("ev-join-processor", "pv-window-store", "ev-pv-window-store", pvStoreWindowDuration, evPvStoreWindowDuration)

new Topology()
  (...)
  .addProcessor("ev-join-processor", evJoinProcessor, "ev-source")
```

The last processor maps compound key `EvPvKey` again into `ClientId`. 
Because client identifier is part of the compound key, repartitioning is done by processor without repartitioning.

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

Finally publish join result to "clickstream.events_enriched" Kafka topic.

``` scala
new Topology()
  (...)
  .addSink("ev-pv-sink", EvPvTopic, "clickstream.events_enriched")
```

If a processor requires access to the store this fact must be registered.
It would be nice to have statically typed Topology API for registration, 
but now if the store is not connected to the processor, runtime exception is thrown during application startup.

``` scala
new Topology()
  (...)
  .addStateStore(pvStore, "pv-window-processor", "ev-join-processor")
  .addStateStore(evPvStore, "ev-join-processor") 
```

Let's count Kafka Streams internal topics overhead for Processor API version.
Wait, there is only one internal topic, for page view join window!

**4k** messages per second and **4MB** traffic-in overhead, not more.
Noticeable difference, if we keep in mind that enrichment results are almost identical to results from DSL version.

## Summary

Dear readers, are you still with me after long lecture with not so easy to digest Scala code?

My final thoughts about Kafka Streams:

* Kafka DSL looks great at first, functional and declarative API sells the product, no doubts.
* Unfortunately Kafka DSL hides a lot of internals which should be exposed via the API 
(stores configuration, join semantics, repartitioning).
* Processor API seems to be more complex and less sexy than DSL.
* But Processor API allows you to create hand-crafted, very efficient stream topologies.
* I did not present any Kafka Streams test (what's the shame - sorry) 
but I think that testing would be easier with Processor API than DSL. 
With DSL it has to be an integration test, processors can be easily unit tested in separation with a few mocks.
* As Scala developer I prefer Processor API than DSL, 
e.g. Scala compiler could not infer KStream generic types. 
* It's a pleasure to work with processor and fluent Topology APIs. 
* I'm really keen on KSQL future, it would be great to get optimized engine like 
[Spark Catalyst](https://databricks.com/blog/2015/04/13/deep-dive-into-spark-sqls-catalyst-optimizer.html) eventually.
* Kafka Streams library is extraordinarily fast and hardware efficient, if you know what you are doing.

As always working code is published on 
[github.com/mkuthan/example-kafkastreams](https://github.com/mkuthan/example-kafkastreams).
The project is configured with [Embedded Kafka](https://github.com/manub/scalatest-embedded-kafka)
and does not require any additional setup. 
Just uncomment either DSL or Processor API version, run main class and observe enriched stream of event on the console.

Enjoy!
