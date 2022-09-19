---
title: "Spark and Spark Streaming unit testing"
date: 2015-03-01
tags: [Apache Spark, Scala, Software Engineering]
header:
    overlay_image: /assets/images/2015-03-01-spark-unit-testing/jakub-skafiriak-AljDaiCbCVY-unsplash.webp
    caption: "[Unsplash](https://unsplash.com/@jakubskafiriak)"
---

When you develop a distributed system, it's crucial to make it easy to test.
Execute tests in a controlled environment, ideally from your IDE.
Long develop-test-develop cycle for complex systems could kill your productivity.
Below you find my testing strategy for Spark and Spark Streaming applications.

## Unit or integration tests, that is the question

Our hypothetical Spark application pulls data from Apache Kafka, apply transformations using 
[RDDs](https://spark.apache.org/docs/latest/api/scala/#org.apache.spark.rdd.RDD) 
and 
[DStreams](https://spark.apache.org/docs/latest/api/scala/#org.apache.spark.streaming.dstream.DStream)
and persist outcomes into Cassandra or ElasticSearch database.
On production Spark application is deployed on YARN or Mesos cluster, and everything is glued with ZooKeeper.
Big picture of the stream processing architecture is presented below:

![](http://yuml.me/diagram/scruffy;dir:LR/class/[Apache%20Kafka]-[Spark%20Streaming],[Spark%20Streaming]-[Cassandra],[Spark%20Streaming]-[Elastic%20Search],[Spark%20Streaming],[Zookeeper{bg:cornsilk}],[YARN%20or%20Mesos%20Cluster{bg:cornsilk}])

Lots of moving parts, not so easy to configure and test.
Even with automated provisioning implemented with Vagrant, Docker and Ansible.
If you can't test everything, test at least the most important part of your application - transformations - implemented with Spark.

Spark claims that it's friendly to unit testing with any popular unit test framework.
To be strict, Spark supports rather lightweight integration testing, not unit testing, IMHO.
But still it's much more convenient to test transformation logic locally, than deploying all parts on YARN.

There is a pull request [SPARK-1751](https://github.com/apache/spark/pull/1751) that adds "unit tests" support for Apache Kafka streams.
Should we follow that way? Embedded ZooKeeper and embedded Apache Kafka are needed, the test fixture is complex and cumbersome.
Perhaps tests would be fragile and hard to maintain. This approach makes sense for Spark core team, they want to test Spark and Kafka integration.

## What should be tested?

Our transformation logic is implemented with Spark, nothing more. 
But how to test the logic so tightly coupled to Spark API (RDD, DStream)?
Let's define how a typical Spark application is organized. 
Our hypothetical application structure looks like this:

1. Initialize `SparkContext` or `StreamingContext`.
2. Create RDD or DStream for given source (e.g: Apache Kafka)
3. Evaluate transformations on RDD or DStream API.
4. Put transformation outcomes (e.g: aggregations) into an external database.

### Context

`SparkContext` and `StreamingContext` could be easily initialized for testing purposes.
Set master URL to `local`, run the operations and then stop context gracefully.

SparkContext initialization:

```scala
class SparkExampleSpec extends FlatSpec with BeforeAndAfter {

  private val master = "local[2]"
  private val appName = "example-spark"

  private var sc: SparkContext = _

  before {
    val conf = new SparkConf()
      .setMaster(master)
      .setAppName(appName)

    sc = new SparkContext(conf)
  }

  after {
    if (sc != null) {
      sc.stop()
    }
  }
  (...)

```

StreamingContext initialization:

```scala 
class SparkStreamingExampleSpec extends FlatSpec with BeforeAndAfter {

  private val master = "local[2]"
  private val appName = "example-spark-streaming"
  private val batchDuration = Seconds(1)
  private val checkpointDir = Files.createTempDirectory(appName).toString

  private var sc: SparkContext = _
  private var ssc: StreamingContext = _

  before {
    val conf = new SparkConf()
      .setMaster(master)
      .setAppName(appName)

    ssc = new StreamingContext(conf, batchDuration)
    ssc.checkpoint(checkpointDir)

    sc = ssc.sparkContext
  }

  after {
    if (ssc != null) {
      ssc.stop()
    }
  }

  (...)
```

### RDD and DStream

The problematic part is how to create RDD or DStream.
For testing purposes it must be simplified to avoid embedded Kafka and ZooKeeper.
Below you can find examples on how to create in-memory RDD and DStream.

In-memory RDD:

```scala
val lines = Seq("To be or not to be.", "That is the question.")
val rdd = sparkContext.parallelize(lines)
```

In-memory DStream:

```scala
val lines = mutable.Queue[RDD[String]]()
val dstream = streamingContext.queueStream(lines)

// append data to DStream
lines += sparkContext.makeRDD(Seq("To be or not to be.", "That is the question."))
```

### Transformation logic

The most important part of our application - transformations logic - must be encapsulated in a separate class or object.
Object is preferred to avoid class serialization overhead. 
Exactly the same code is used by the application and by the test.

WordCount.scala

```scala
case class WordCount(word: String, count: Int)

object WordCount {
  def count(lines: RDD[String], stopWords: Set[String]): RDD[WordCount] = {
    val words = lines.flatMap(_.split("\\s"))
      .map(_.strip(",").strip(".").toLowerCase)
      .filter(!stopWords.contains(_)).filter(!_.isEmpty)

    val wordCounts = words.map(word => (word, 1)).reduceByKey(_ + _).map {
      case (word: String, count: Int) => WordCount(word, count)
    }

    val sortedWordCounts = wordCounts.sortBy(_.word)

    sortedWordCounts
  }
}
```

## Spark test

Now it's time to implement our first test for WordCount transformation.
The code of the test is very straightforward and easy to read.
Single point of truth, the best documentation of your system, always up-to-date.

```scala
"Shakespeare most famous quote" should "be counted" in {
    Given("quote")
    val lines = Array("To be or not to be.", "That is the question.")

    Given("stop words")
    val stopWords = Set("the")

    When("count words")
    val wordCounts = WordCount.count(sc.parallelize(lines), stopWords).collect()

    Then("words counted")
    wordCounts should equal(Array(
      WordCount("be", 2),
      WordCount("is", 1),
      WordCount("not", 1),
      WordCount("or", 1),
      WordCount("question", 1),
      WordCount("that", 1),
      WordCount("to", 2)))
  }
```

## Spark Streaming test

Spark Streaming transformations are much more complex to test.
Full control over the clock is needed to manually manage batches, slides and windows.
Without a controlled clock you would end up with complex tests with many `Thread.sleeep` calls.
And the test execution would take ages.
The only downside is that you will not have extra time for coffee during test execution.

Spark Streaming provides necessary abstraction over system clock, `ManualClock` class.
Unfortunately `ManualClock` class is declared as package private. Some hack is needed.
The wrapper presented below, is an adapter for the original `ManualClock` class but without access restriction.

ClockWrapper.scala

```scala
package org.apache.spark.streaming

import org.apache.spark.streaming.util.ManualClock

class ClockWrapper(ssc: StreamingContext) {

  def getTimeMillis(): Long = manualClock().currentTime()

  def setTime(timeToSet: Long) = manualClock().setTime(timeToSet)

  def advance(timeToAdd: Long) = manualClock().addToTime(timeToAdd)

  def waitTillTime(targetTime: Long): Long = manualClock().waitTillTime(targetTime)

  private def manualClock(): ManualClock = {
    ssc.scheduler.clock.asInstanceOf[ManualClock]
  }

}
```

Now the Spark Streaming test can be implemented in an efficient way.
The test doesn't have to wait for the system clock and the test is implemented with millisecond precision.
You can easily test your windowed scenario from the very beginning to very end.
With the given\when\then structure you should be able to understand tested logic without further explanations.

```scala
"Sample set" should "be counted" in {
  Given("streaming context is initialized")
  val lines = mutable.Queue[RDD[String]]()

  var results = ListBuffer.empty[Array[WordCount]]

  WordCount.count(ssc.queueStream(lines), windowDuration, slideDuration) { (wordsCount: RDD[WordCount], time: Time) =>
    results += wordsCount.collect()
  }

  ssc.start()

  When("first set of words queued")
  lines += sc.makeRDD(Seq("a", "b"))

  Then("words counted after first slide")
  clock.advance(slideDuration.milliseconds)
  eventually(timeout(1 second)) {
    results.last should equal(Array(
      WordCount("a", 1),
      WordCount("b", 1)))
  }

  When("second set of words queued")
  lines += sc.makeRDD(Seq("b", "c"))

  Then("words counted after second slide")
  clock.advance(slideDuration.milliseconds)
  eventually(timeout(1 second)) {
    results.last should equal(Array(
      WordCount("a", 1),
      WordCount("b", 2),
      WordCount("c", 1)))
  }

  When("nothing more queued")

  Then("word counted after third slide")
  clock.advance(slideDuration.milliseconds)
  eventually(timeout(1 second)) {
    results.last should equal(Array(
      WordCount("a", 0),
      WordCount("b", 1),
      WordCount("c", 1)))
  }

  When("nothing more queued")

  Then("word counted after fourth slide")
  clock.advance(slideDuration.milliseconds)
  eventually(timeout(1 second)) {
    results.last should equal(Array(
      WordCount("a", 0),
      WordCount("b", 0),
      WordCount("c", 0)))
  }
}
```

One comment to `Eventually` trait usage.
The trait is needed because Spark Streaming is a multithreaded application, and results aren't computed immediately.
I found that 1 second timeout is enough for Spark Streaming to calculate the results.
The timeout isn't related to batch, slide or window duration.

## Summary

The complete, working project is published on [GitHub](https://github.com/mkuthan/example-spark).
You can clone/fork the project and do some experiments by yourself.

I hope that Spark committers expose `ManualClock` for others, eventually.
Control of time is necessary for efficient Spark Streaming application testing.
