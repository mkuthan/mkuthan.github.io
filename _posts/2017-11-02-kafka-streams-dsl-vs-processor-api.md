---
title: "Kafka Streams DSL vs processor API"
date: 2017-11-02
categories: [Apache Kafka, Kafka Streams, Scala, Performance]
tagline: ""
header:
    overlay_image: /assets/images/mitchell-kmetz-5a3gboOg9qc-unsplash.webp
    overlay_filter: 0.2
---

[Kafka Streams](https://docs.confluent.io/current/streams/index.html) is a Java library 
for building real-time, highly scalable, fault-tolerant, distributed applications. 
The library is fully integrated with [Kafka](https://kafka.apache.org/documentation/) and leverages 
Kafka producer and consumer semantics (e.g: partitioning, rebalancing, data retention and compaction).
What is unique, the only dependency to run Kafka Streams application is a running Kafka cluster.
Even local state stores are backed by Kafka topics to make the processing fault-tolerant - brilliant!

Kafka Streams provides all necessary stream processing primitives like one-record-at-a-time processing, 
event time processing, windowing support and local state management. 
Application developers can choose from three different Kafka Streams APIs: DSL, Processor or KSQL.

* [Kafka Streams DSL](https://docs.confluent.io/current/streams/developer-guide.html#kafka-streams-dsl) 
(Domain Specific Language) - recommended way for most users 
because business logic can be expressed in a few lines of code.
All stateless and stateful transformations are defined using declarative, 
functional programming style (filter, map, flatMap, reduce, aggregate operations).
Kafka Stream DSL encapsulates most of the stream processing complexity,
but unfortunately it also hides many useful knobs and switches. 

* [Kafka Processor API](https://docs.confluent.io/current/streams/developer-guide.html#processor-api) 
provides a low level, imperative way to define stream processing logic.
At first sight Processor API could look hostile but finally gives much more flexibility to developers.
With this blog post I would like to demonstrate that hand-crafted stream processors might be a magnitude more efficient
than a naive implementation using Kafka DSL.

* [KSQL](https://www.confluent.io/product/ksql/) 
is a promise that stream processing could be expressed by anyone using SQL as the language.
It offers an easy way to express stream processing transformations as an alternative to writing 
an application in a programming language such as Java.
Moreover, processing transformations written in SQL-like language can be highly optimized
by the execution engine without any developer effort. 
KSQL was released recently, and it is still at a very early development stage.

In the first part of this blog post I'll define a simple but still realistic business problem to solve.
Then you will learn how to implement this use case with Kafka Stream DSL 
and how much the processing performance is affected by this naive solution.
At this moment you could stop reading and scale-up Kafka cluster ten times to fulfill business requirements 
or you could continue reading and learn how to optimize the processing with low level Kafka Processor API.

## Business Use Case

Let's imagine a web based e-commerce platform with fabulous recommendation and advertisement systems.
Every client during the visit gets personalized recommendations and advertisements,
the conversion is extraordinarily high and the platform earns additional profits from advertisers.
To build comprehensive recommendation models, 
such a system needs to know everything about clients' traits and their behavior.

To make it possible, e-commerce platforms report all clients activities as an unbounded stream 
of page views and events.
Every time the client enters a web page, a so-called page view is sent to the Kafka cluster. 
A page view contains web page attributes like request URI, referrer URI, user agent, active A/B experiments
and many more.
In addition to page view all important actions are reported as custom events, e.g: search, add to cart or checkout.
To get a complete view of the activity stream, collected events need to be enriched with data from page views.

## Data Model

Because most of the processing logic is built within the context of a given client, 
page views and events are evenly partitioned on Kafka topics by the client identifier.

``` scala
type ClientId = String
case class ClientKey(clientId: ClientId)

val bob = ClientKey("bob")
val jim = ClientKey("jim")
```

Page view and event structures are different so messages are published to separate Kafka topics
using ingestion time as the event time. 
Our system should not rely on pageview or event creation time due to high client clock variance.
The topic key is always `ClientKey` and value is either `Pv` or `Ev` presented below.
For better examples readability page view and event payload is defined as a simplified single value field.
Events are uniquely identified by `pvId` and `evId` pair, `pvId` could be a random identifier, `evId` 
a sequence number.

``` scala
type PvId = String
type EvId = String

case class Pv(pvId: PvId, value: String)
case class Ev(evId: EvId, value: String, pvId: PvId) 
```

Enriched results `EvPv` is published to output Kafka topic using `ClientKey` as message key.
This topic is then consumed directly by advertisement and recommendation systems.

``` scala
case class EvPv(evId: EvId, evValue: String, pvId: Option[PvId], pvValue: Option[String])
```

## Example Scenario

For client "bob" the following page views and events are collected by the system.

``` scala
// Bob enters main page
ClientKey("bob"), Pv("pv0", "/")

// A few impression events collected almost immediately
ClientKey("bob"), Ev("ev0", "show header", "pv0")
ClientKey("bob"), Ev("ev1", "show ads", "pv0")
ClientKey("bob"), Ev("ev2", "show recommendation", "pv0")

// There is also single duplicated event, welcome to distributed world
ClientKey("bob"), Pv("ev1", "show ads", "pv0")

// A dozen seconds later Bob clicks on one of the offers presented on the main page
ClientKey("bob"), Pv("ev3", "click recommendation", "pv0")

// Out of order event collected before page view on the offer page 
ClientKey("bob"), Ev("ev0", "show header", "pv1")

// Offer page view
ClientKey("bob"), Pv("pv1", "/offer?id=1234")

// An impression event published almost immediately after page view
ClientKey("bob"), Ev("ev1", "show ads", "pv1")

// Late purchase event, Bob took short coffee break before the final decision
ClientKey("bob"), Ev("ev2", "add to cart", "pv1")
```

For the above click-stream the following enriched events output stream is expected.

``` scala
// Events from main page without duplicates

ClientKey("bob"), EvPv("ev0", "show header", "pv0", "/")
ClientKey("bob"), EvPv("ev1", "show ads", "pv0", "/")
ClientKey("bob"), EvPv("ev2", "show recommendation", "pv0", "/")
ClientKey("bob"), EvPv("ev3", "click recommendation", "pv0", "/")

// Events from offer page, somehow incomplete due to streaming semantics limitations

// early event
ClientKey("bob"), EvPv("ev0", "show header", None, None)
ClientKey("bob"), EvPv("ev1", "show ads", "pv1", "/offer?id=1234")

// late event
ClientKey("bob"), EvPv("ev2", "add to cart", None, None) 
```

## Kafka Stream DSL

Now we are ready to implement the above use case with recommended the Kafka Streams DSL. 
The code could be optimized, but I would like to present the canonical way of using DSL
without exploring DSL internals.
All examples are implemented using the latest Kafka Streams 1.0.0 version.

Create two input streams for page views and events 
connected to "clickstream.events" and "clickstream.page_views" Kafka topics.

``` scala
val builder = new StreamsBuilder()

val evStream = builder.stream[ClientKey, Ev]("clickstream.events")
val pvStream = builder.stream[ClientKey, Pv]("clickstream.page_views")
```

Repartition topics by client and page view identifiers `PvKey` 
as a prerequisite to join events with page view.
Method `selectKey` sets a new key for every input record,
and marks the derived stream for repartitioning.

``` scala
case class PvKey(clientId: ClientId, pvId: PvId)

val evToPvKeyMapper: KeyValueMapper[ClientKey, Ev, PvKey] =
  (clientKey, ev) => PvKey(clientKey.clientId, ev.pvId)

val evByPvKeyStream = evStream.selectKey(evToPvKeyMapper)

val pvToPvKeyMapper: KeyValueMapper[ClientKey, Pv, PvKey] =
  (clientKey, pv) => PvKey(clientKey.clientId, pv.pvId)

val pvByPvKeyStream = pvStream.selectKey(pvToPvKeyMapper)
```

Join event with page-view streams by selected previously `PvKey`, 
left join is used because we are interested also in events without matched page view.
Every incoming event is enriched by matched page view into `EvPv` structure.
 
The join window duration is set to a reasonable 10 minutes.
It means that Kafka Streams will look for messages in "event" and "page view" sides of the join 
10 minutes in the past and 10 minutes in the future (using event time, not wall-clock time).
Because we are not interested in late events out of the defined window, 
the retention is 2 times longer than the window, to hold events from the past and the future.
If you are interested why 1 milliseconds needs to be added to the retention, 
please ask Kafka Streams authors, not me ;)

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

Now it's time to fight with duplicated enriched events. 
Duplicates come from the unreliable nature of the network between the client browser and our system.
Most real-time processing pipelines in advertising and recommendation systems are counting events,
so duplicates in the enriched click-stream could cause inaccuracies.

The most straightforward deduplication method is to compare incoming events with state of previously processed events.
If the event has been already processed it should be skipped.

Unfortunately DSL does not provide a "deduplicate" method out-of-the-box but similar logic might be implemented with 
"reduce" operation.

First we need to define deduplication window.
The deduplication window can be much shorter than the join window, 
we do not expect duplicates more than 10 seconds between each other.

``` scala
val deduplicationWindowDuration = 10 seconds

val deduplicationRetention = deduplicationWindowDuration.toMillis * 2 + 1
val deduplicationWindow = TimeWindows.of(deduplicationWindowDuration.toMillis).until(deduplicationRetention)
```

Joined stream needs to be repartitioned again by compound key `EvPvKey` composed of 
client, page view and event identifiers.
This key will be used to decide if `EvPv` is a duplicate or not.
Next, the stream is grouped by selected key into KGroupedStream and
deduplicated with reduce function, where the first observed event wins.

``` scala
case class EvPvKey(clientId: ClientId, pvId: PvId, evId: EvId)

val evPvToEvPvKeyMapper: KeyValueMapper[PvKey, EvPv, EvPvKey] =
  (pvKey, evPv) => EvPvKey(pvKey.clientId, pvKey.pvId, evPv.evId)

val evPvByEvPvKeyStream = evPvStream.selectKey(evPvToEvPvKeyMapper)

val evPvDeduplicator: Reducer[EvPv] =
  (evPv1, _) => evPv1


val deduplicatedStream = evPvByEvPvKeyStream
  .groupByKey()
  .reduce(evPvDeduplicator, deduplicationWindow, "evpv-store")
  .toStream()
```

This deduplication implementation is debatable, due to "continue stream" semantics of KTable/KStream.
Reduce operation creates KTable, and this KTable is transformed again into KStream of continuous updates of the same key.
It could lead to duplicates again if the update frequency is higher than the inverse of the deduplication window period.
For the 10 seconds deduplication window the updates should not be emitted more often than every 10 seconds but
lower updates frequency leads to higher latency.
The updates frequency is controlled globally using "cache.max.bytes.buffering" and "commit.interval.ms" 
Kafka Streams properties. 
See reference documentation for details:
[Record caches in the DSL](https://docs.confluent.io/current/streams/developer-guide.html#streams-developer-guide-memory-management).

I did not find another way to deduplicate events with DSL, please let me know if better implementation exists.

In the last stage the stream needs to be repartitioned again by client id 
and published to "clickstream.events_enriched" Kafka topic for downstream subscribers.
In the same step mapper gets rid of the windowed key produced by the windowed reduce function.

``` scala
val evPvToClientKeyMapper: KeyValueMapper[Windowed[EvPvKey], EvPv, ClientId] =
  (windowedEvPvKey, _) => windowedEvPvKey.key.clientId

val finalStream = deduplicatedStream.selectKey(evPvToClientKeyMapper)

finalStream.to("clickstream.events_enriched")
```


## Under The Hood

Kafka Stream DSL is quite descriptive, isn't it? 
Especially developers with strong functional programming skills appreciate the overall design.
But you will shortly see how much unexpected traffic to Kafka cluster is generated during runtime.

I like numbers so let's estimate the traffic, 
based on real clickstream ingestion platform I develop on daily basis:

* 1 kB - average page view size 
* 600 B - average event size
* 4k - page views / second 
* 20k - events / second

It gives 24k msgs/s and 16MB/s traffic-in total, the traffic easily handled even by small Kafka clusters.

When a stream of data is repartitioned Kafka Streams creates additional intermediate topic
and publishes on the topic whole traffic partitioned by selected key. 
To be more precise it happens twice in our case, for repartitioned page views and events before joining.
We need to add 24k msgs/s and 16MB/s more traffic-in to the calculation.

When streams of data are joined using a window, Kafka Streams sends both sides of the join
to two intermediate topics again. Even if you don't need fault tolerance,
logging into Kafka cannot be disabled using DSL.
You cannot also get rid of the window for "this" side of the join (window for events), more about it later on. 
Add 24k msgs/s and 16MB/s more traffic-in to the calculation again.

To deduplicate events, joined stream goes again into Kafka Streams intermediate topic.
Add 20k msgs/s and (1kB + 1.6kB) * 20k = 52MB/s more traffic-in to the calculation again.

The last repartitioning by client identifier adds 20k msgs/s and 52MB/s more traffic-in.

Finally, instead of **24k** msgs/s and **16MB/s** traffic-in we have got
**112k** msgs/s and **152MB** traffic-in.
And I did not even count traffic from internal topics replication and standby replicas 
[recommended for resiliency](https://docs.confluent.io/current/streams/developer-guide.html#recommended-configuration-parameters-for-resiliency). 

Be aware that this is calculation for simple join of events and pages views generated by 
local e-commerce platform in central Europe country (~20M clients). 
I could also easily imagine much more complex stream topology, with tens of repartitions, joins and aggregations.

If you are not careful, your Kafka Streams application could easily kill your Kafka cluster.
At least our application did it once. Application deployed on 10 [Mesos](http://mesos.apache.org/) 
nodes (4CPU, 4GB RAM) almost killed Kafka cluster deployed also on 10 physical machines (32CPU, 64GB RAM, SSD).
Application was started after some time of inactivity and processed 3 hours of retention in 5 minutes
(yep, it's a well known vulnerability until 
[KIP-13](https://cwiki.apache.org/confluence/display/KAFKA/KIP-13+-+Quotas) is open).

## Kafka Processor API

Now it's time to check the Processor API and figure out how to optimize our stream topology.

Create the sources from input topics "clickstream.events" and "clickstream.page_views".

``` scala
new Topology()
  .addSource("ev-source", "clickstream.events")
  .addSource("pv-source", "clickstream.page_views")
```

Because we need to join an incoming event with the collected page view in the past,
create a processor which stores the page-view in the windowed store. 
The processor puts observed page views into the window store for joining in the next processor.
The processed page views do not even need to be forwarded to downstream.

``` scala
class PvWindowProcessor(val pvStoreName: String) extends AbstractProcessor[ClientKey, Pv] {

  private lazy val pvStore =
    context().getStateStore(pvStoreName).asInstanceOf[WindowStore[ClientKey, Pv]]

  override def process(key: ClientKey, value: Pv): Unit =
    pvStore.put(key, value)
}
```

Store for page views is configured with the same size of window and retention.
This store is configured to keep duplicates due to the fact 
that the key is a client id not page view id (retainDuplicates parameter).
Because the join window is typically quite long (minutes) the store should be fault tolerant (logging enabled).
Even if one of the stream instances fails, 
another one will continue processing with a persistent window state built by a failed node, cool!
Finally, the internal kafka topic can be easily configured using loggingConfig map 
(replication factor, number of partitions, etc.).

``` scala
val pvStoreWindowDuration = 10 minutes
 
val retention = pvStoreWindowDuration.toMillis
val window = pvStoreWindowDuration.toMillis
val segments = 3
val retainDuplicates = true

val loggingConfig = Map[String, String]()

val pvWindowStore = Stores.windowStoreBuilder(
  Stores.persistentWindowStore("pv-window-store", retention, segments, window, retainDuplicates),
  ClientKeySerde,
  PvSerde
).withLoggingEnabled(loggingConfig)
```

The first optimization you could observe is that in our scenario only one window store is created - for page views.
The window store for events is not needed, if the page-view is collected by the system after the event it does not trigger a new join.

Add page view processor to the topology and connect with page view source upstream.

``` scala
val pvWindowProcessor: ProcessorSupplier[ClientKey, Pv] =
  () => new PvWindowProcessor("pv-window-store")
  
new Topology()
  (...)
  .addProcessor("pv-window-processor", pvWindowProcessor, "pv-source")
```

Now, it's time for the event and page view join processor, the heart of the topology.
It seems to be complex but this processor also deduplicates joined streams using `evPvStore`.

``` scala
class EvJoinProcessor(
  val pvStoreName: String,
  val evPvStoreName: String,
  val joinWindow: FiniteDuration,
  val deduplicationWindow: FiniteDuration
) extends AbstractProcessor[ClientKey, Ev] {

  import scala.collection.JavaConverters._

  private lazy val pvStore =
    context().getStateStore(pvStoreName).asInstanceOf[WindowStore[ClientKey, Pv]]

  private lazy val evPvStore =
    context().getStateStore(evPvStoreName).asInstanceOf[WindowStore[EvPvKey, EvPv]]

  override def process(key: ClientKey, ev: Ev): Unit = {
    val timestamp = context().timestamp()
    val evPvKey = EvPvKey(key.clientId, ev.pvId, ev.evId)

    if (isNotDuplicate(evPvKey, timestamp, deduplicationWindow)) {
      val evPv = storedPvs(key, timestamp, joinWindow)
        .find { pv =>
          pv.pvId == ev.pvId
        }
        .map { pv =>
          EvPv(ev.evId, ev.value, Some(pv.pvId), Some(pv.value))
        }
        .getOrElse {
          EvPv(ev.evId, ev.value, None, None)
        }

      context().forward(evPvKey, evPv)
      evPvStore.put(evPvKey, evPv)
    }
  }

  private def isNotDuplicate(evPvKey: EvPvKey, timestamp: Long, deduplicationWindow: FiniteDuration) =
    evPvStore.fetch(evPvKey, timestamp - deduplicationWindow.toMillis, timestamp).asScala.isEmpty

  private def storedPvs(key: ClientKey, timestamp: Long, joinWindow: FiniteDuration) =
    pvStore.fetch(key, timestamp - joinWindow.toMillis, timestamp).asScala.map(_.value)
  }
```

First processor performs a lookup for previously joined `PvEv` by `PvEvKey`.
If `PvEv` is found the processing is skipped because `EvPv` has been already processed.

Next, try to match page view to event using simple filter `pv.pvId == ev.pvId`.
We don't need any repartitioning to do that, only get all page views from given client 
and join with event in the processor itself. 
It should be very efficient because every client generates up to a hundred page-views in 10 minutes.
If there is no matched page view in the configured window, 
`EvPv` without page view details is forwarded to the downstream.

Perceptive reader noticed that the processor also changes the key from `ClientId` to `EvPvKey`
for deduplication purposes. 
Everything is still within the given client context without the need for any repartitioning.
This is possible due to the fact that the new key is more detailed than the original one.

As before, a windowed store for deduplication needs to be configured.
Because deduplication is done in a very short window (10 seconds or so),
the logging to backed internal Kafka topic is disabled at all.
If one of the stream instances fails, we could get some duplicates during this short window, not a big deal.

``` scala
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

Add the join processor to the topology and connect with the event source upstream.

``` scala
val evJoinProcessor: ProcessorSupplier[ClientKey, Ev] =
  () => new EvJoinProcessor("pv-window-store", "ev-pv-window-store", pvStoreWindowDuration, evPvStoreWindowDuration)

new Topology()
  (...)
  .addProcessor("ev-join-processor", evJoinProcessor, "ev-source")
```

The last processor maps compound key `EvPvKey` again into `ClientId`. 
Because the client identifier is already a part of the compound key, 
mapping is done by the processor without the need for further repartitioning.

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

Finally publish join results to "clickstream.events_enriched" Kafka topic.

``` scala
new Topology()
  (...)
  .addSink("ev-pv-sink", EvPvTopic, "clickstream.events_enriched")
```

If a processor requires access to the store this fact must be registered.
It would be nice to have statically typed Topology API for registration, 
but now if the store is not connected to the processor, 
or is connected to the wrong store,
a runtime exception is thrown during application startup.

``` scala
new Topology()
  (...)
  .addStateStore(pvStore, "pv-window-processor", "ev-join-processor")
  .addStateStore(evPvStore, "ev-join-processor") 
```

Let's count Kafka Streams internal topics overhead for the Processor API version.
Wait, there is only one internal topic, for page view join window!
It gives 4k messages per second and 4MB traffic-in overhead, not more.

**28k** instead of **112k** messages per second and **20MB** instead of **152MB** traffic-in in total. 
There is a noticeable difference between Processor API and DSL topology versions, 
especially if we keep in mind that enrichment results are almost identical to results from the DSL version.

## Summary

Dear readers, are you still with me after a long lecture with not so easy to digest Scala code?
I hope so :)

My final thoughts about Kafka Streams:

* Kafka DSL looks great at first, functional and declarative API sells the product, no doubts.
* Unfortunately Kafka DSL hides a lot of internals which should be exposed via the API 
(stores configuration, join semantics, repartitioning) - see 
[KIP-182](https://cwiki.apache.org/confluence/display/KAFKA/KIP-182%3A+Reduce+Streams+DSL+overloads+and+allow+easier+use+of+custom+storage+engines).
* Processor API seems to be more complex and less sexy than DSL.
* But Processor API allows you to create hand-crafted, very efficient stream topologies.
* I did not present any Kafka Streams test (what's the shame - I'm sorry) 
but I think testing would be easier with Processor API than DSL. 
With DSL it has to be an integration test, processors can be easily unit tested in separation with a few mocks.
* As Scala developer I prefer Processor API over DSL, 
e.g. Scala compiler could not infer KStream generic types. 
* It's a pleasure to work with processor and fluent Topology APIs. 
* I'm really keen on KSQL future, it would be great to get an optimized engine like 
[Spark Catalyst](https://databricks.com/blog/2015/04/13/deep-dive-into-spark-sqls-catalyst-optimizer.html) eventually.
* Finally, Kafka Streams library is extraordinarily fast and hardware efficient, if you know what you are doing.

As always, working code is published on 
[https://github.com/mkuthan/example-kafkastreams](https://github.com/mkuthan/example-kafkastreams).
The project is configured with [Embedded Kafka](https://github.com/manub/scalatest-embedded-kafka)
and does not require any additional setup. 
Just uncomment either DSL or Processor API version, run main class and observe enriched stream of events on the console.

Enjoy!
