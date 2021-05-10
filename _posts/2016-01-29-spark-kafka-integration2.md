---
title: "Spark and Kafka integration patterns, part 2"
date: 2016-01-29
categories: [spark, kafka, scala]
---

In the [world beyond batch](https://www.oreilly.com/ideas/the-world-beyond-batch-streaming-101), 
streaming data processing is a future of dig data. 
Despite of the streaming framework using for data processing, tight integration with replayable data source like Apache Kafka is often required.
The streaming applications often use Apache Kafka as a data source, or as a destination for processing results.

Apache Spark distribution has built-in support for reading from Kafka, but surprisingly does not offer any
integration for sending processing result back to Kafka.
This blog post aims to fill this gap in the Spark ecosystem.

In the [first part](http://mkuthan.github.io/blog/2015/08/06/spark-kafka-integration1/) of the series
you learned how to manage Kafka producer using Scala lazy evaluation feature
and how to reuse single Kafka producer instance on Spark executor.

In this blog post you will learn how to publish stream processing results to Apache Kafka in reliable way.
First you will learn how Kafka Producer is working, 
how to configure Kafka producer and how to setup Kafka cluster to achieve desired reliability.
In the second part of the blog post, 
I will present how to implement convenient library for sending continuous sequence of RDDs 
([DStream](https://spark.apache.org/docs/latest/api/scala/#org.apache.spark.streaming.dstream.DStream)) 
to Apache Kafka topic, as easy as in the code snippet below.

``` scala
// enable implicit conversions
import KafkaDStreamSink._

// send dstream to Kafka
dstream.sendToKafka(kafkaProducerConfig, topic)
```

## Quick introduction to Kafka

Kafka is a distributed, partitioned, replicated message broker.
Basic architecture knowledge is a prerequisite to understand Spark and Kafka integration challenges.
You can safely skip this section, if you are already familiar with Kafka concepts. 

For convenience I copied essential terminology definitions directly from Kafka 
[documentation](http://kafka.apache.org/documentation.html#introduction):
 
* Kafka maintains feeds of messages in categories called topics.
* We'll call processes that publish messages to a Kafka topic producers.
* We'll call processes that subscribe to topics and process the feed of published messages consumers.
* Kafka is run as a cluster comprised of one or more servers each of which is called a broker.

So, at a high level, producers send messages over the network to the Kafka cluster which in turn serves them up to consumers like this:

![](http://kafka.apache.org/images/producer_consumer.png)

This is a bare minimum you have to know but I really encourage you to read Kafka reference manual thoroughly.

## Kafka producer API

First we need to know how Kafka producer is working.
Kafka producer exposes very simple API for sending messages to Kafka topics. 
The most important methods from `KafkaProducer` class are listed below:

KafkaProducer API

``` java
j.u.c.Future<RecordMetadata> send(ProducerRecord<K,V> record)
j.u.c.Future<RecordMetadata> send(ProducerRecord<K,V> record, Callback callback)
void flush()
void close()
```

The `send()` methods asynchronously send a key-value record to a topic and will return immediately once the record has been stored in the buffer of records waiting to be sent.
This kind of API is not very convenient for developers, but is crucial to achieve high throughput and low latency.

If you want to ensure that request has been completed, you can invoke blocking `get()` on the future returned by the `send()` methods.
The main drawback of calling `get()` is a huge performance penalty because it disables batching effectively. 
You can not expect high throughput and low latency if the execution is blocked on every message and every single message needs to be sent separately.

Fully non-blocking usage requires use of the callback. The callback will be invoked when the request is complete. 
Note that callback is executed in Kafka producer I/O thread so should not block the caller, the callback must be as lightweight as possible.
The callback must be also properly synchronized due to [Java memory model](https://en.wikipedia.org/wiki/Java_memory_model).

If the Kafka producer caller does not check result of the `send()` method using future or callback, 
it means that if Kafka producer crashed all messages from the internal Kafka producer buffer will be lost. 
This is the first, very important element of any integration library with Kafka, 
we should expect callback handling to avoid data lost and achieve good performance.

The `flush()` method makes all buffered messages ready to send, and blocks on the completion of the requests associated with these messages.
The `close()` method is like the `flush()` method but also closes the producer.

The `flush()` method could be very handy if the Streaming framework wants to ensure that all messages have been sent before processing next part of the stream.
With `flush()` method streaming framework is able to flush the messages to Kafka brokers to simulate commit behaviour.

Method `flush()` was added in Kafka 0.9 release ([KIP-8](https://cwiki.apache.org/confluence/display/KAFKA/KIP-8+-+Add+a+flush+method+to+the+producer+API)). 
Before Kafka 0.9, the only safe and straightforward way to flush messages from Kafka producer internal buffer was to close the producer.

## Kafka configuration

If the message must be reliable published on Kafka cluster, Kafka producer and Kafka cluster needs to be configured with care.
It needs to be done independently of chosen streaming framework.

Kafka producer buffers messages in memory before sending.
When our memory buffer is exhausted, Kafka producer must either stop accepting new records (block) or throw errors.
By default Kafka producer blocks and this behavior is legitimate for stream processing. 
The processing should be delayed if Kafka producer memory buffer is full and could not accept new messages.
Ensure that `block.on.buffer.full` Kafka producer configuration property is set.

With default configuration, when Kafka broker (leader of the partition) receive the message, store the message in memory and immediately send acknowledgment to Kafka producer.
To avoid data loss the message should be replicated to at least one replica (follower). 
Only when the follower acknowledges the leader, the leader acknowledges the producer.
 
This guarantee you will get with `ack=all` property in Kafka producer configuration. 
This guarantees that the record will not be lost as long as at least one in-sync replica remains alive.

But this is not enough. The minimum number of replicas in-sync must be defined.
You should configure `min.insync.replicas` property for every topic. 
I recommend to configure at least 2 in-sync replicas (leader and one follower).
If you have datacenter with two zones, I also recommend to keep leader in the first zone and 2 followers in the second zone.
This configuration guarantees that every message will be stored in both zones.

We are almost done with Kafka cluster configuration.
When you set `min.insync.replicas=2` property, the topic should be replicated with factor 2 + N. 
Where N is the number of brokers which could fail, and Kafka producer will still be able to publish messages to the cluster.
I recommend to configure replication factor 3 for the topic (or more).

With replication factor 3, the number of brokers in the cluster should be at least 3 + M.
When one or more brokers are unavailable, you will get underreplicated partitions state of the topics. 
With more brokers in the cluster than replication factor, you can reassign underreplicated partitions and achieve fully replicated cluster again.
I recommend to build the 4 nodes cluster at least for topics with replication factor 3.

The last important Kafka cluster configuration property is `unclean.leader.election.enable`. 
It should be disabled (by default it is enabled) to avoid unrecoverable exceptions from Kafka consumer. 
Consider the situation when the latest committed offset is N,
but after leader failure, the latest offset on the new leader is M < N.
M < N because the new leader was elected from the lagging follower (not in-sync replica).
When the streaming engine ask for data from offset N using Kafka consumer, it will get an exception because the offset N does not exist yet.
Someone will have to fix offsets manually.

So the minimal recommended Kafka setup for reliable message processing is:

* 4 nodes in the cluster
* `unclean.leader.election.enable=false` in the brokers configuration
* replication factor for the topics - 3
* `min.insync.replicas=2` property in topic configuration
* `ack=all` property in the producer configuration
* `block.on.buffer.full=true` property in the producer configuration

With the above setup your configuration should be resistant to single broker failure, 
and Kafka consumers will survive new leader election. 

You could also take look at `replica.lag.max.messages` and `replica.lag.time.max.ms` properties 
for tuning when the follower is removed from ISR by the leader. 
But this is out of this blog post scope.

## How to expand Spark API?

After this not so short introduction, we are ready to disassembly 
[integration library](https://github.com/mkuthan/example-spark-kafka) for Spark Streaming and Apache Kafka.
First `DStream` needs to be somehow expanded to support new method `sendToKafka()`. 

``` scala
dstream.sendToKafka(kafkaProducerConfig, topic)
```

In Scala, the only way to add methods to existing API, is to use an implicit conversion feature. 
 
``` scala
object KafkaDStreamSink {

  import scala.language.implicitConversions

  implicit def createKafkaDStreamSink(dstream: DStream[KafkaPayload]): KafkaDStreamSink = {
    new KafkaDStreamSink(dstream)
  }

}
```

Whenever Scala compiler finds call to non-existing method `sendToKafka()` on `DStream` class, 
the stream will be implicitly wrapped into `KafkaDStreamSink` class, 
where method `sendToKafka` is finally defined.
To enable implicit conversion for `DStream` add the import statement to your code, that's all.

``` scala 
import KafkaDStreamSink._
```


## How to send to Kafka in reliable way?

Let's check how `sendToKafka()` method is defined step by step, this is the core part of the integration library.

``` scala
class KafkaDStreamSink(dstream: DStream[KafkaPayload]) {

  def sendToKafka(config: Map[String, String], topic: String): Unit = {
    dstream.foreachRDD { rdd =>
      rdd.foreachPartition { records =>
        // send records from every partition to Kafka
      }
    }
  }
```

There are two loops, first on wrapped `dstream` and second on `rdd` for every partition.
Quite standard pattern for Spark programming model.
Records from every partition are ready to be sent to Kafka topic by Spark executors.
The destination topic name is given explicitly as the last parameter of the `sendToKafka()` method.


First step in the inner loop is getting Kafka producer instance from the `KafkaProducerFactory`.

``` scala
rdd.foreachPartition { records =>
  val producer = KafkaProducerFactory.getOrCreateProducer(config)
  (...)
```

The factory creates only single instance of the producer for any given producer configuration.
If the producer instance has been already created, the existing instance is returned and reused.
Kafka producer caching is crucial for the performance reasons, 
because establishing a connection to the cluster takes time. 
It is a much more time consuming operation than opening plain socket connection, 
as Kafka producer needs to discover leaders for all partitions.
Please refer to [first part](http://mkuthan.github.io/blog/2015/08/06/spark-kafka-integration1/) of this blog post 
and `KafkaProducerFactory`
[source](https://github.com/mkuthan/example-spark-kafka/blob/master/src/main/scala/org/mkuthan/spark/KafkaProducerFactory.scala) 
for more details about the factory implementation.

For debugging purposes logger and Spark task context are needed.

``` scala
import org.apache.spark.TaskContext
import org.slf4j.LoggerFactory
(...)

rdd.foreachPartition { records =>
  val producer = KafkaProducerFactory.getOrCreateProducer(config)
  
  val context = TaskContext.get 
  val logger = Logger(LoggerFactory.getLogger(classOf[KafkaDStreamSink]))
  (...)
```

You could use any logging framework but the logger itself has to be defined in the foreachPartition loop
to avoid weird serialization issues.
Spark task context will be used to get current partition identifier.
I don't like static call for getting task context, but this is an official way to do that.
See pull request [SPARK-5927](https://github.com/apache/spark/pull/5927) for more details. 


Before we go further, Kafka producer callback for error handling needs to be introduced.

KafkaDStreamSinkExceptionHandler:

``` scala
import java.util.concurrent.atomic.AtomicReference
import org.apache.kafka.clients.producer.Callback

class KafkaDStreamSinkExceptionHandler extends Callback {

  private val lastException = new AtomicReference[Option[Exception]](None)

  override def onCompletion(metadata: RecordMetadata, exception: Exception): Unit = 
    lastException.set(Option(exception))

  def throwExceptionIfAny(): Unit = 
    lastException.getAndSet(None).foreach(ex => throw ex)

}
```

Method `onCompletion()` of the callback is called when the message sent to the Kafka cluster has been acknowledged.
Exactly one of the callback arguments will be non-null, `metadata` or `exception`.
`KafkaDStreamSinkExceptionHandler` class keeps last exception registered by the callback (if any).
The client of the callback is able to rethrow registered exception using `throwExceptionIfAny()` method.
Because `onCompletion()` and `throwExceptionIfAny()` methods are called from different threads,
last exception has to be kept in thread-safe data structure `AtomicReference`.

Finally we are ready to send records to Kafka using created callback.

``` scala
rdd.foreachPartition { records =>
  val producer = KafkaProducerFactory.getOrCreateProducer(config)
  
  val context = TaskContext.get 
  val logger = Logger(LoggerFactory.getLogger(classOf[KafkaDStreamSink]))
  
  val callback = new KafkaDStreamSinkExceptionHandler
  
  logger.debug(s"Send Spark partition: ${context.partitionId} to Kafka topic: $topic")
  val metadata = records.map { record =>
    callback.throwExceptionIfAny()
    producer.send(new ProducerRecord(topic, record.key.orNull, record.value), callback)
  }.toList
```

First the callback is examined for registered exception. 
If one of the previous record could not be sent, the exception is propagated to Spark framework.
If any redelivery policy is needed it should be configured on Kafka producer level. 
Look at Kafka producer configuration properties `retries` and `retry.backoff.ms`.
Finally Kafka producer metadata are collected and materialized by calling `toList()` method.
At this moment, Kafka producer starts sending records in background I/O thread. 
To achieve high throughput Kafka producer sends records in batches.
       
Because we want to achieve natural back pressure for our stream processing,
next batch needs to be blocked until records from current batch are really acknowledged by the Kafka brokers.
So for each collected metadata (Java `j.u.c.Future`), method `get()` is called to ensure that record has been sent to the brokers.

``` scala
rdd.foreachPartition { records =>
  val producer = KafkaProducerFactory.getOrCreateProducer(config)
  
  val context = TaskContext.get 
  val logger = Logger(LoggerFactory.getLogger(classOf[KafkaDStreamSink]))
  
  val callback = new KafkaDStreamSinkExceptionHandler
  
  logger.debug(s"Send Spark partition: ${context.partitionId} to Kafka topic: $topic")
  val metadata = records.map { record =>
    callback.throwExceptionIfAny()
    producer.send(new ProducerRecord(topic, record.key.orNull, record.value), callback)
  }.toList

  logger.debug(s"Flush Spark partition: ${context.partitionId} to Kafka topic: $topic")
  metadata.foreach { metadata => metadata.get() }
```

As long as records sending was started moment ago, it is likelihood that records have been already sent
and `get()` method does not block. 
However if the `get()` call is blocked, it means that there are unsent messages in the internal Kafka producer buffer 
and the processing should be blocked as well.

Finally `sendToKafka()` method should propagate exception recorded by the callback (if any).
Complete method is presented below for reference.

``` scala
def sendToKafka(config: Map[String, String], topic: String): Unit = {
  dstream.foreachRDD { rdd =>
    rdd.foreachPartition { records =>
      val producer = KafkaProducerFactory.getOrCreateProducer(config)

      val context = TaskContext.get
      val logger = Logger(LoggerFactory.getLogger(classOf[KafkaDStreamSink]))

      val callback = new KafkaDStreamSinkExceptionHandler

      logger.debug(s"Send Spark partition: ${context.partitionId} to Kafka topic: $topic")
      val metadata = records.map { record =>
        callback.throwExceptionIfAny()
        producer.send(new ProducerRecord(topic, record.key.orNull, record.value), callback)
      }.toList

      logger.debug(s"Flush Spark partition: ${context.partitionId} to Kafka topic: $topic")
      metadata.foreach { metadata => metadata.get() }

      callback.throwExceptionIfAny()
    }
  }
}
```

The method is not very complex but there are a few important elements
if you don't want to lose processing results and if you need back pressure mechanism:

* Method `sendToKafka()` should fail fast if record could not be sent to Kafka. Don't worry Spark will execute failed task again.
* Method `sendToKafka()` should block Spark processing if Kafka producer slows down.
* Method `sendToKafka()` should flush records buffered by Kafka producer explicitly, to avoid data loss.
* Kafka producer needs to be reused by Spark executor to avoid connection to Kafka overhead.
* Kafka producer needs to be explicitly closed when Spark shutdowns executors to avoid data loss.

## Summary

The complete, working project is published on 
[https://github.com/mkuthan/example-spark-kafka](https://github.com/mkuthan/example-spark-kafka). 
You can clone/fork the project and do some experiments by yourself.

There is also alternative library developed by Cloudera 
[spark-kafka-writer](https://github.com/cloudera/spark-kafka-writer)
emerged from closed pull request [SPARK-2994](https://github.com/apache/spark/pull/2994).
Unfortunately at the time of this writing, 
the library used obsolete Scala Kafka producer API and did not send processing results in reliable way.

I hope that some day we will find reliable, mature library for sending processing result to Apache Kafka
in the official Spark distribution.
