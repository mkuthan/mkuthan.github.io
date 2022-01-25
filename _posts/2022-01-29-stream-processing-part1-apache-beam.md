---
title: "Stream Processing - Part 1 - Apache Beam"
date: 2022-01-29
categories: [stream processing, apache beam, scala]
---

This is a very first part of [stream processing](/categories/stream-processing/) series.
From the series you will learn how to develop and test stateful streaming pipelines.

I'm going to start with examples implemented with [Apache Beam](https://beam.apache.org/) and [Scio](https://spotify.github.io/scio/),
then check and compare capabilities in other frameworks like [Apache Flink](https://flink.apache.org), 
[Structured Spark Streaming](https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html) or 
[Kafka Streams](https://kafka.apache.org/documentation/streams/).

Expect well-crafted code samples verified by test and some theory fundamental to stream processing if needed:

* event time vs. processing time
* latency vs. completeness
* unbounded data windowing
* watermarks, triggers, accumulation, punctuation and more


## Word Count

Let's start with "Word Count" on unbounded stream of lines.
The pipeline should produce the cardinality of each observed word.
Because this is a streaming pipeline, the results are materialized periodically on every minute.

For example, the following stream of lines:
```
(...)
00:00:00 -> "foo bar"
00:00:30 -> "baz baz"
00:01:00 -> "foo bar"
00:01:30 -> "bar foo"
(...)
```

Should be partitioned by event time into same-length, non-overlapping chunks of data:
```
(...)
00:01:00 -> (foo, 1) (bar, 1), (baz, 2)
00:02:00 -> (foo, 2) (bar, 2)
(...)
```

I'm fan of TDD so the implementation of the aggregation method will be disclosed at the end of this blog post.
As for now the following method signature should be enough to understand all test scenarios:

```scala
def wordCountInFixedWindow(
    lines: SCollection[String],
    windowDuration: Duration
): SCollection[(String, Long)]
```
* Method takes unbounded stream of lines (Scio `SCollection` or Beam `PCollection`)
* Parameter `windowDuration` defines length of the fixed window
* Result is defined as word with cardinality tuple
* You can not find any footprint of event time in the signature

The event time is passed implicitly by the framework. 
This is a common pattern for streaming systems, because every part of the system must be always event-time aware 
the event-time is transported outside the main payload. 
Moreover, the event-time is constantly advanced during journey through the pipeline 
so keeping event-time as a part of the payload does not make sense.

Let's move to the test scenarios.

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

Nothing fancy except one mystery method: `advanceWatermarkToInfinity()`.
In a nutshell the method tells: "all lines in the input stream have been observed, calculate the results please".

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

* All result elements get end-of-window time "00:00:01" as new event-time
* It means that every fixed window in the pipeline introduces additional latency

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
}
```

* If there is no input lines for given fixed window, no results are produced

## Late Data

I'm glad that you are still there, the most interesting part of this blog post starts here.

Late data is an integral part of every streaming application. 
Imagine that our stream comes from mobile application and someone is on a train that has hit long tunnel somewhere in the Alps ...

Streaming pipeline needs to materialize results in a timely manner, how long the pipeline should wait for data?
If on 99th percentile latency is 3 seconds, it does not make any sense to wait for outliers late by minutes or hours. 
For unbounded data this is always heuristic calculation, 
the streaming engine continuously estimates time "X" where all input data with event-time less than "X" have been observed.
The time "X" is called watermark.

Fortunately it's quite easy to write fully deterministic test scenario for late data. 
Good streaming frameworks (like Apache Beam) provide watermark programmatic control.

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

* After two on-time events the watermark is programmatically advanced to the end of the first one-minute window
* The results for the first window are materialized as before
* Then late event is observed, it should be included in the window "00:01:00" but it is silently dropped!

## Late Data Under Allowed Lateness (Discarded)

What if late data must be included in the final calculation?
As a pipeline developer we should define allowed lateness.

```scala
val DefaultWindowDuration = Duration.standardMinutes(1L)

"Late words under allowed lateness" should "be aggregated in late pane" in runWithContext { sc =>
    val words = testStreamOf[String]
      .addElementsAt("00:00:00", "foo bar")
      .addElementsAt("00:00:30", "baz baz")
      .advanceWatermarkTo("00:01:00")
      .addElementsAt("00:00:40", "foo foo") // late event under allowed lateness
      .advanceWatermarkToInfinity()
    
    val allowedLateness = Duration.standardSeconds(30)
    val results = wordCountInFixedWindow(sc.testStream(words), DefaultWindowDuration, allowedLateness)
    
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
* Late result gets end-of-window time "00:00:01" as new event-time, exactly as on-time results
* There are special assertions to ensure that aggregation comes from on-time or late pane

## Late Data Under Allowed Lateness (Accumulated)

In the previous example the results from on-time pane are discarded and not used for late pane.
If the pipeline writes result into idempotent sink we could accumulate on-time into late pane, and update incomplete result when late data arrives. 

```scala
"Late words under allowed lateness" should "be aggregated and accumulated in late pane" in runWithContext { sc =>
    val words = testStreamOf[String]
      .addElementsAt("00:00:00", "foo bar")
      .addElementsAt("00:00:30", "baz baz")
      .advanceWatermarkTo("00:01:00")
      .addElementsAt("00:00:40", "foo foo") // late event under allowed lateness
      .advanceWatermarkToInfinity()
    
    val allowedLateness = Duration.standardSeconds(30)
    val accumulationMode = AccumulationMode.ACCUMULATING_FIRED_PANES
    val results = wordCountInFixedWindow(sc.testStream(words), DefaultWindowDuration, allowedLateness, accumulationMode)
    
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

* Accumulation mode is set to "ACCUMULATING_FIRED_PANES" instead of default "DISCARDING_FIRED_PANES"
* Result in late pane counts "foo" from both panes
* Be aware, accumulation means more resources utilized by the streaming framework

## Summary

Are you interested how the aggregation is implemented actually ([full source code](https://github.com/mkuthan/example-streaming)?
Streaming pipelines are a magnitude more complex to test than batch pipelines :)

```scala
def wordCountInFixedWindow(
      lines: SCollection[String],
      windowDuration: Duration,
      allowedLateness: Duration = Duration.ZERO,
      accumulationMode: AccumulationMode = AccumulationMode.DISCARDING_FIRED_PANES
): SCollection[(String, Long)] = {
    val windowOptions = WindowOptions(
      allowedLateness = allowedLateness,
      accumulationMode = accumulationMode
    )
    
    lines
      .flatMap(line => line.split("\\s+"))
      .withFixedWindows(duration = windowDuration, options = windowOptions)
      .countByValue
}
```
Key takeaways:

* To aggregate unbounded stream the data must be partitioned by event-time
* Every aggregation introduce latency, event-time advances through the pipeline
* Late date is inevitable for streaming pipelines
* Watermark handling is the most important feature of any streaming framework

I hope that you enjoy the first blog post in [stream processing](/categories/stream-processing/) series.
Put a comment below if you want more :)