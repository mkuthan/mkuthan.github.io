---
title: "Stream Processing - Part 1"
date: 2022-01-29
categories: [stream processing, apache beam, scala]
---

This is the very first part of [stream processing](/categories/stream-processing/) blog post series.
From the series you will learn how to develop and test stateful streaming data pipelines.

## Overview

I'm going to start with examples implemented with [Apache Beam](https://beam.apache.org/) and [Scio](https://spotify.github.io/scio/),
then check and compare capabilities of other frameworks like [Apache Flink](https://flink.apache.org), 
[Structured Spark Streaming](https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html) or 
[Kafka Streams](https://kafka.apache.org/documentation/streams/).

Expect well-crafted code samples verified by integration tests, and stream processing fundamental theory if it's absolutely necessary:

* event time vs. processing time
* latency vs. completeness
* windowing, watermarks, triggers, accumulation, punctuation and more

## Word Count

Let's start with *Word Count* on unbounded stream of text lines.
The pipeline produces the cardinality of each observed word.
Because this is a streaming pipeline, the results are materialized periodically - in this case every minute.

For example, the following stream of lines:
```
(...)
00:00:00 -> "foo bar"
00:00:30 -> "baz baz"
00:01:00 -> "foo bar"
00:01:30 -> "bar foo"
(...)
```

Should be aggregated and partitioned by event time into same-length, non-overlapping chunks of data:
```
(...)
00:01:00 -> (foo, 1) (bar, 1) (baz, 2)
00:02:00 -> (foo, 2) (bar, 2)
(...)
```

I'm a fan of TDD as a learning technique, so the implementation of the aggregation method will not be disclosed until the very end of this blog post.
As for now, the following method signature should be enough to understand all presented test scenarios:

```scala
def wordCountInFixedWindow(
  lines: SCollection[String],
  windowDuration: Duration
): SCollection[(String, Long)]
```

* Method takes payload - unbounded stream of lines (Scio [SCollection](https://spotify.github.io/scio/api/com/spotify/scio/values/SCollection.html), 
  Beam [PCollection](https://beam.apache.org/documentation/programming-guide/#pcollections),
  Flink [Dataset](https://javadoc.io/static/org.apache.flink/flink-java/1.14.3/org/apache/flink/api/java/DataSet.html),
  Spark [Dataset](https://spark.apache.org/docs/latest/api/java/org/apache/spark/sql/Dataset.html) or 
  Kafka [KStream](https://kafka.apache.org/31/javadoc/org/apache/kafka/streams/kstream/KStream.html))
* Parameter `windowDuration` defines length of the fixed non-overlapping window
* Result is defined as tuple: (word, cardinality)

There is no trace of event time in the signature, WTF?

The event time is passed implicitly which is a common pattern for streaming frameworks.
Because every part of the system must be always event-time aware, the event-time is transported outside the payload. 
Moreover, the event-time is constantly advanced during data journey through the pipeline 
so keeping event-time as a part of the payload does not make sense.

Let's move to the first test scenario.

## No Input

If the input stream is empty the aggregation should not emit any results.

```scala
val DefaultWindowDuration = Duration.standardMinutes(1L)

"Words aggregate" should "be empty for empty stream" in runWithContext { sc =>
  val words = testStreamOf[String].advanceWatermarkToInfinity()
  
  val results = wordCountInFixedWindow(sc.testStream(words), DefaultWindowDuration)
  
  results should beEmpty
}
```

Nothing fancy, except one mystery method: `advanceWatermarkToInfinity()`.
In a nutshell the method tells: *all lines in the input stream have been already observed, calculate the results please*.
Unclear? I'm fully with you but don't worry, you will learn more about watermarks later on.

## Single Window

The scenario where input lines are aggregated into words in single fixed one-minute window.

```scala
val DefaultWindowDuration = Duration.standardMinutes(1L)

"Words" should "be aggregated into single fixed window" in runWithContext { sc =>
  val words = testStreamOf[String]
    .addElementsAt("00:00:00", "foo bar")
    .addElementsAt("00:00:30", "baz baz")
    .advanceWatermarkToInfinity()
  
  val results = wordCountInFixedWindow(sc.testStream(words), DefaultWindowDuration)
  
  results.withTimestamp should containInAnyOrderAtWindowTime(Seq(
    ("00:01:00", ("foo", 1L)),
    ("00:01:00", ("bar", 1L)),
    ("00:01:00", ("baz", 2L))
  ))
}
```

* All result elements get end-of-window time "00:00:01" as a new event-time
* It means that every fixed window in the pipeline introduces additional latency
* Longer window requires more resources allocated by streaming runtime

## Single Window (Latest Timestamp Combiner)

You might be tempted to materialize the aggregate with timestamp of latest observed element instead of end-of-window time.
Look at the next scenario:

```scala
"Words" should "be aggregated into single fixed window with latest timestamp" in runWithContext { sc =>
  val words = testStreamOf[String]
    .addElementsAt("00:00:00", "foo bar")
    .addElementsAt("00:00:30", "baz baz")
    .advanceWatermarkToInfinity()

  val results = wordCountInFixedWindow(
    sc.testStream(words), 
     DefaultWindowDuration, 
     timestampCombiner = TimestampCombiner.LATEST)

  results.withTimestamp should containInAnyOrderAtTime(Seq(
    ("00:00:00", ("foo", 1L)),
    ("00:00:00", ("bar", 1L)),
    ("00:00:30", ("baz", 2L)),
  ))
}
```

* All result elements get the latest timestamp of the words instead of end-of-window time
* But the overall latency of the pipeline is exactly the same

The results will be materialized at the same processing time but the time skew between processing and event time will be larger.
Let me explain on the example, assume that average time skew for the input is 10 seconds (events are delayed by 10 seconds).
For end-of-window timestamp combiner, the time skew after aggregation will be also 10 seconds or a little more.
For latest timestamp combiner, the skew after aggregation will be 60 + 10 seconds for `foo` and 30 + 10 seconds for `baz`.
So don't cheat with the latest timestamp combiner for fixed window, you can not travel back in time :)

## Consecutive Windows

The scenario when the lines are aggregated into words for two consecutive fixed one-minute windows.

```scala
val DefaultWindowDuration = Duration.standardMinutes(1L)

"Words" should "be aggregated into consecutive fixed windows" in runWithContext { sc =>
  val words = testStreamOf[String]
    .addElementsAt("00:00:00", "foo bar")
    .addElementsAt("00:00:30", "baz baz")
    .addElementsAt("00:01:00", "foo bar")
    .addElementsAt("00:01:30", "bar foo")
    .advanceWatermarkToInfinity()
  
  val results = wordCountInFixedWindow(sc.testStream(words), DefaultWindowDuration)
  
  results.withTimestamp should containInAnyOrderAtWindowTime(Seq(
    ("00:01:00", ("foo", 1L)),
    ("00:01:00", ("bar", 1L)),
    ("00:01:00", ("baz", 2L)),
    ("00:02:00", ("foo", 2L)),
    ("00:02:00", ("bar", 2L)),
  ))
}
```

* Again, all result elements get end-of-window time
* The aggregates from each fixed window are independent
* It means that aggregation results from the first window are discarded after materialization and framework is able to free allocated resources

## Non-Consecutive Windows

The scenario when the lines are aggregated into words for two non-consecutive fixed one-minute windows.

```scala
val DefaultWindowDuration = Duration.standardMinutes(1L)

"Words" should "be aggregated into non-consecutive fixed windows" in runWithContext { sc =>
  val words = testStreamOf[String]
    .addElementsAt("00:00:00", "foo bar")
    .addElementsAt("00:00:30", "baz baz")
    .addElementsAt("00:02:00", "foo bar")
    .addElementsAt("00:02:30", "bar foo")
    .advanceWatermarkToInfinity()
  
  val results = wordCountInFixedWindow(sc.testStream(words), DefaultWindowDuration)
  
  results.withTimestamp should containInAnyOrderAtWindowTime(Seq(
    ("00:01:00", ("foo", 1L)),
    ("00:01:00", ("bar", 1L)),
    ("00:01:00", ("baz", 2L)),
    ("00:03:00", ("foo", 2L)),
    ("00:03:00", ("bar", 2L)),
  ))
  
  results.withTimestamp should inOnTimePane("00:01:00", "00:02:00") {
    beEmpty
  }
}
```

* If there is no input lines for given period, no results are produced

It is quite problematic trait of the streaming pipelines for the frameworks.
How to recognize if the pipeline is stale from the situation when everything works smoothly but there is no data for some period of time?
It becomes especially important for joins, if there is no incoming data in one stream it can not block the pipeline forever.


## Late Data

I'm glad that you are still there, the most interesting part of this blog post starts here.

Late data is an inevitable part of every streaming application. 
Imagine that our stream comes from mobile application and someone is on a train that has hit long tunnel somewhere in the Alps ...

Streaming pipeline needs to materialize results in a timely manner, how long the pipeline should wait for data?
If 99<sup>th</sup> percentile latency is 3 seconds, it does not make any sense to wait for outliers late by minutes or hours. 
For unbounded data this is always heuristic calculation, 
the streaming framework continuously estimates time "X" where all input data with event-time less than "X" have been already observed.
The time "X" is called **watermark**. 
The watermark calculation algorithm determines quality of the streaming framework. 
With better heuristics you will get lower overall latency and more precise late data handling. 

Fortunately it's quite easy to write fully deterministic test scenario for late data. 
Good streaming frameworks (like Apache Beam) provide watermark programmatic control for testing purposes.

```scala
val DefaultWindowDuration = Duration.standardMinutes(1L)

"Late words" should "be silently dropped" in runWithContext { sc =>
  val words = testStreamOf[String]
    .addElementsAt("00:00:00", "foo bar")
    .addElementsAt("00:00:30", "baz baz")
    .advanceWatermarkTo("00:01:00")
    .addElementsAt("00:00:40", "foo") // late event
    .advanceWatermarkToInfinity()
  
  val results = wordCountInFixedWindow(sc.testStream(words), DefaultWindowDuration)
  
  results.withTimestamp should containInAnyOrderAtWindowTime(Seq(
    ("00:01:00", ("foo", 1L)),
    ("00:01:00", ("bar", 1L)),
    ("00:01:00", ("baz", 2L)),
  ))
}
```

* After "foo bar" and "baz baz" on-time events, the watermark is programmatically advanced to the end of the first one-minute window
* As an effect of updated watermark, the results for the first window are materialized.
* Then late event "foo" is observed, it should be included in the results of window "00:01:00" but it is silently dropped!

With high quality heuristic watermark it should be rare situation that watermark is advanced too early. 
But as a developer you have to take into account this kind of situation.

## Late Data Under Allowed Lateness (Discarded)

What if late data must be included in the final calculation?
**Allowed lateness** comes to the rescue.

```scala
val DefaultWindowDuration = Duration.standardMinutes(1L)

"Late words under allowed lateness" should "be aggregated in late pane" in runWithContext { sc =>
  val words = testStreamOf[String]
    .addElementsAt("00:00:00", "foo bar")
    .addElementsAt("00:00:30", "baz baz")
    .advanceWatermarkTo("00:01:00")
    .addElementsAt("00:00:40", "foo foo") // late event under allowed lateness
    .advanceWatermarkToInfinity()
  
  val results = wordCountInFixedWindow(
    sc.testStream(words), DefaultWindowDuration, 
    allowedLateness = Duration.standardSeconds(30))
  
  results.withTimestamp should inOnTimePane("00:00:00", "00:01:00") {
    containInAnyOrderAtWindowTime(Seq(
      ("00:01:00", ("foo", 1L)),
      ("00:01:00", ("bar", 1L)),
      ("00:01:00", ("baz", 2L)),
    ))
  }
  
  results.withTimestamp should inLatePane("00:00:00", "00:01:00") {
    containSingleValueAtWindowTime(
      "00:01:00", ("foo", 2L)
    )
  }
}
```

* The late event under allowed lateness is included in the result
* Late result gets end-of-window time of window starting at "00:00:00" as new event-time, exactly as on-time results
* There are special assertions to ensure that aggregation comes from on-time or late pane

Late data propagates through the pipeline.
Most probably late result from one step is still considered late in downstream steps.
Configure allowed lateness consistently for all pipeline steps.

## Late Data Under Allowed Lateness (Accumulated)

In the previous example the results from *on-time pane* are discarded and not used for the *late pane*.
If the pipeline writes result into idempotent sink we could accumulate on-time into late pane, and update incomplete result when late data arrives. 

```scala
"Late words under allowed lateness" should "be aggregated and accumulated in late pane" in runWithContext { sc =>
  val words = testStreamOf[String]
    .addElementsAt("00:00:00", "foo bar")
    .addElementsAt("00:00:30", "baz baz")
    .advanceWatermarkTo("00:01:00")
    .addElementsAt("00:00:40", "foo foo") // late event under allowed lateness
    .advanceWatermarkToInfinity()
  
  val results = wordCountInFixedWindow(
    sc.testStream(words), DefaultWindowDuration, 
    allowedLateness = Duration.standardSeconds(30), 
    accumulationMode = AccumulationMode.ACCUMULATING_FIRED_PANES)
  
  results.withTimestamp should inOnTimePane("00:00:00", "00:01:00") {
    containInAnyOrderAtWindowTime(Seq(
      ("00:01:00", ("foo", 1L)),
      ("00:01:00", ("bar", 1L)),
      ("00:01:00", ("baz", 2L))
    ))
  }
  
  results.withTimestamp should inLatePane("00:00:00", "00:01:00") {
    containSingleValueAtWindowTime(
      "00:01:00", ("foo", 3L)
    )
  }
}
```

* Accumulation mode is set to `ACCUMULATING_FIRED_PANES` instead of default `DISCARDING_FIRED_PANES`
* Result in *late pane* counts `foo` from both panes
* Be aware, accumulation means more resources utilized by the streaming framework

## Summary

Are you interested how the aggregation is implemented actually?
Feel free to inspect [source code](https://github.com/mkuthan/example-streaming) to get the whole picture of the examples.

```scala
def wordCountInFixedWindow(
  lines: SCollection[String],
  windowDuration: Duration,
  allowedLateness: Duration = Duration.ZERO,
  accumulationMode: AccumulationMode = AccumulationMode.DISCARDING_FIRED_PANES,
  timestampCombiner: TimestampCombiner = TimestampCombiner.END_OF_WINDOW
): SCollection[(String, Long)] = {
  val windowOptions = WindowOptions(
    allowedLateness = allowedLateness,
    accumulationMode = accumulationMode,
    timestampCombiner = timestampCombiner
  )
  
  lines
    .flatMap(line => line.split("\\s+"))
    .withFixedWindows(duration = windowDuration, options = windowOptions)
    .countByValue
}
```

Key takeaways:

* Streaming pipelines are magnitude more complex to test than batch pipelines.
* To aggregate unbounded stream the data must be partitioned by event-time.
* Every aggregation introduce latency, event-time typically advances through the pipeline.
* Late data is inevitable for streaming pipelines.
* Watermark handling is the most important feature of any streaming framework.

I hope that you enjoy the first blog post in [stream processing](/categories/stream-processing/) series.
Let me know what do you think as a comment below.
