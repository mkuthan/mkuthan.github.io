---
title: "Stream Processing - Part 2"
date: 2022-02-18
categories: [stream processing, apache beam, scala]
tagline: Apache Beam - dynamic aggregation
header:
    overlay_image: /assets/images/forest-simon-yQ1IDlOe-6k-unsplash.webp
    overlay_filter: 0.2
---

This is the second part of the [stream processing](/categories/stream-processing/) blog post series.
In the first [part](/blog/2022/01/28/stream-processing-part1/) I presented aggregations in fixed non-overlapping window.
Now you will learn dynamic aggregations in data-driven window, 
when the size of the window depends on the input data themselves instead of predefined time based pattern.
At the end I also show the complex triggering policy to get speculative results for the latency sensitive pipelines.

## User sessions

One of the most popular dynamic aggregations is a session window. 
The single session captures user activities within some period of time followed by the gap of inactivity.

As an example, the following stream of user's activities is ingested from hypothetical mobile e-commerce application:

```
"00:00:00" -> "open app"
"00:01:30" -> "show product"
"00:03:00" -> "add to cart"
"00:13:00" -> "checkout"
"00:13:10" -> "close app"
```

The activities should be aggregated into a single session if the maximum allowed gap between the activities is equal or greater than 10 minutes
(the greatest gap is between "add to cart" and "checkout" events):

```
00:23:10 -> "open app", "show product", "add to cart", "checkout", "close app"
```

Or into two independent sessions, if the allowed gap between activities is shorter, e.g. for 5 minutes gap:

```
00:08:00 -> "open app", "show product", "add to cart"
00:18:10 -> "checkout", "close app"
```

This is over-simplified aggregation example just for this blog academic purposes, notice that it does not scale at all. 
All user activities during the session must fit into memory of the single machine.
In real-world scenario the actions should be reduced into scalable [algebraic](https://en.wikipedia.org/wiki/Algebraic_structure) data structures, e.g: 

* various counters of the session like the number of unique visited products
* number of deals
* click-through rate  
* conversion rate

I'm going to follow the TDD technique again, so the implementation of the aggregation method will not be disclosed until the very end of the blog post.
As for now, the following method signature should be enough to understand all tests' scenarios:

```scala
type User = String
type Activity = String

def activitiesInSessionWindow(
  userActions: SCollection[(User, Activity)],
  gapDuration: Duration,
): SCollection[(User, Iterable[Activity])]
```

## No activities, no sessions

If the input stream of activities is empty the aggregation should not emit any session.

```scala
"No activities" should "create empty session" in runWithContext { sc =>
  val activities = testStreamOf[(User, Activity)].advanceWatermarkToInfinity()

  val results = activitiesInSessionWindow(sc.testStream(activities), TenMinutesGap)

  results should beEmpty
}
```

Nothing fancy, except one mystery method: `advanceWatermarkToInfinity()`.
In a nutshell the method tells: *all activities in the input stream have been already observed, calculate the sessions please*.
Unclear? If so, check the first [part](/blog/2022/01/28/stream-processing-part1/) of the series.

## Single session

The scenario where all activities are aggregated into a single session.

```scala
"Activities" should "be aggregated into single session" in runWithContext { sc =>
  val activities = testStreamOf[(User, Activity)]
    .addElementsAtTime("00:00:00", ("joe", "open app"))
    .addElementsAtTime("00:01:00", ("joe", "close app"))
    .advanceWatermarkToInfinity()

  val results = activitiesInSessionWindow(sc.testStream(activities), TenMinutesGap)

  results.withTimestamp should inOnTimePane("00:00:00", "00:11:00") {
    containSingleValueAtTime("00:10:59.999", 
      ("joe", Iterable("open app", "close app")))
  }
}
```

* The result's window is defined as the half-open interval *[00:00:00, 00:11:00)*
* End boundary gets *00:11:00*, ten minutes (session gap duration) after the last observed activity event time
* The result gets end-of-window time *00:10:59.999* as a new event-time
* It means that every session window in the pipeline introduces additional latency
* Longer window requires more resources allocated by streaming runtime for keeping the state

## Two simultaneous sessions

The scenario in which activities from two clients are aggregated into two simultaneous sessions.

```scala
"Activities from two clients" should "be aggregated into two simultaneous sessions" in runWithContext { sc =>
  val activities = testStreamOf[(User, Activity)]
    .addElementsAtTime("00:00:00", ("joe", "open app"))
    .addElementsAtTime("00:00:00", ("ben", "open app"))
    .addElementsAtTime("00:01:00", ("joe", "close app"))
    .addElementsAtTime("00:01:30", ("ben", "close app"))
    .advanceWatermarkToInfinity()

  val results = activitiesInSessionWindow(sc.testStream(activities), TenMinutesGap)

  results.withTimestamp should inOnTimePane("00:00:00", "00:11:00") {
    containSingleValueAtTime("00:10:59.999", 
      ("joe", Iterable("open app", "close app")))
  }

  results.withTimestamp should inOnTimePane("00:00:00", "00:11:30") {
    containSingleValueAtTime("00:11:29.999", 
      ("ben", Iterable("open app", "close app")))
  }
}
```

* Again, all result elements get end-of-window time
* The activities from each user are aggregated into the independent, simultaneous sessions

## Single session with out-of-order activities

Out of order messages are the everyday life in the distributed systems.
The aggregation function should order the activities within the session.

```scala
"Out-of-order activities" should "be aggregated into single session" in runWithContext { sc =>
  val activities = testStreamOf[(User, Activity)]
    .addElementsAtTime("00:01:00", ("joe", "close app"))
    .addElementsAtTime("00:00:00", ("joe", "open app"))
    .advanceWatermarkToInfinity()

  val results = activitiesInSessionWindow(sc.testStream(activities), TenMinutesGap)

  results.withTimestamp should inOnTimePane("00:00:00", "00:11:00") {
    containSingleValueAtTime("00:10:59.999", ("joe", 
      Iterable("open app", "close app")))
  }
}
```

* The activities in the session are ordered by activity event time
* The result gets end-of-window time, ten minutes after the oldest event time 
* The oldest event do not have to be not necessarily the latest one

## Continues long-lasting session

Let's check if the session is preserved for activities span for a longer period of time but still under allowed gap.

```scala
"Continuous activities" should "be aggregated into single session" in runWithContext { sc =>
  val activities = testStreamOf[(User, Activity)]
    .addElementsAtTime("00:00:00", ("joe", "open app"))
    .addElementsAtTime("00:01:30", ("joe", "show product"))
    .addElementsAtTime("00:03:00", ("joe", "add to cart"))
    .addElementsAtTime("00:09:30", ("joe", "checkout"))
    .addElementsAtTime("00:13:10", ("joe", "close app"))
    .advanceWatermarkToInfinity()

  val results = activitiesInSessionWindow(sc.testStream(activities), TenMinutesGap)

  results.withTimestamp should inOnTimePane("00:00:00", "00:23:10") {
    containSingleValueAtTime(
      "00:23:09.999",
      ("joe", Iterable("open app", "show product", "add to cart", "checkout", "close app")))
  }
}
```

* As expected the single long session is produced, there is no larger gap than 10 minutes in the activities stream
* Result element gets end-of-window time *00:23:09.999*, ten minutes more than the last observed event time

## Interrupted long-lasting session

What if the event time gap is larger than the allowed gap for the session window?

```scala
"Interrupted activities" should "be aggregated into two sessions" in runWithContext { sc =>
  val activities = testStreamOf[(User, Activity)]
    .addElementsAtTime("00:00:00", ("joe", "open app")
    .addElementsAtTime("00:01:30", ("joe", "show product"))
    .addElementsAtTime("00:03:00", ("joe", "add to cart"))
    .addElementsAtTime("00:13:00", ("joe", "checkout"))
    .addElementsAtTime("00:13:10", ("joe", "close app"))
    .advanceWatermarkToInfinity()

  val results = activitiesInSessionWindow(sc.testStream(activities), TenMinutesGap)

  results.withTimestamp should inOnTimePane("00:00:00", "00:13:00") {
    containSingleValueAtTime(
      "00:12:59.999",
      ("joe", Iterable("open app", "show product", "add to cart")))
  }

  results.withTimestamp should inOnTimePane("00:13:00", "00:23:10") {
    containSingleValueAtTime("00:23:09.999", ("joe", Iterable("checkout", "close app")))
  }
}
```

* Two independent session windows are produced
* Result element always gets end-of-window time, ten minutes more than the last observed event time for the session

## Late activity

We have slowly moved into the more interesting scenarios. What happens if the late activity will be observed in the stream?

```scala
"Late activity" should "be silently discarded" in runWithContext { sc =>
  val activities = testStreamOf[(User, Activity)]
    .addElementsAtTime("00:00:00", ("joe", "open app"))
    .addElementsAtTime("00:01:30", ("joe", "show product"))
    .advanceWatermarkTo("00:13:00")
    .addElementsAtTime("00:03:00", ("joe", "add to cart")) // late event
    .addElementsAtTime("00:09:30", ("joe", "checkout"))
    .addElementsAtTime("00:13:10", ("joe", "close app"))
    .advanceWatermarkToInfinity()

  val results = activitiesInSessionWindow(sc.testStream(activities), TenMinutesGap)

  results.withTimestamp should inOnTimePane("00:00:00", "00:11:30") {
    containSingleValueAtTime("00:11:29.999", ("joe", Iterable("open app", "show product")))
  }

  results.withTimestamp should inWindow("00:00:00", "00:13:00") {
    beEmpty
  }

  results.withTimestamp should inOnTimePane("00:09:30", "00:23:10") {
    containSingleValueAtTime("00:23:09.999", ("joe", Iterable("checkout", "close app")))
  }
}
```

* Late event is simply discarded, there is no result produces for window *[00:00:00, 00:13:00)*!
* The first session is produces as expected when watermark has passed allowed gap after the last observed event

Did you guess that the second session should have only the "close app" event? 
Are you wrong as you can see, the "checkout" activity at *00:09:30* has been counted as well :)
The "checkout" activity at *00:09:30* has started a new session window, from "00:09:30" to "00:19:30" (ten minutes gap).
As long as watermark is held on *00:13:00* all events after *00:03:00* would be aggregated into a new session window as well.

## Late activity within allowed lateness (Fired Panes Discarded)

How late events would be handled if the late activity is observed under the allowed lateness?

```scala
"Late activity within allowed lateness" should "be aggregated into late pane" in runWithContext { sc =>
  val activities = testStreamOf[(User, Activity)]
    .addElementsAtTime("00:00:00", ("joe", "open app"))
    .addElementsAtTime("00:01:30", ("joe", "show product"))
    .advanceWatermarkTo("00:13:00")
    .addElementsAtTime("00:03:00", ("joe", "add to cart")) // late event within allowed lateness
    .addElementsAtTime("00:09:30", ("joe", "checkout"))
    .addElementsAtTime("00:13:10", ("joe", "close app"))
    .advanceWatermarkToInfinity()

  val results = activitiesInSessionWindow(
    sc.testStream(activities),
    TenMinutesGap,
    allowedLateness = Duration.standardMinutes(5))

  results.withTimestamp should inOnTimePane("00:00:00", "00:11:30") {
    containSingleValueAtTime("00:11:29.999", ("joe", Iterable("open app", "show product")))
  }

  results.withTimestamp should inLatePane("00:00:00", "00:13:00") {
    containSingleValueAtTime("00:12:59.999", ("joe", Iterable("add to cart")))
  }

  results.withTimestamp should inOnTimePane("00:00:00", "00:23:10") {
    containSingleValueAtTime("00:23:09.999", ("joe", Iterable("checkout", "close app")))
  }
}
```

* Nothing special to the first session window
* There is an additional session produced in the late pane, it contains only the late event but the window starts at "00:00:00"!
* Because the late event has closed the gap, the second session also starts at "00:00:00"!

But why the activities from the beginning of the stream has not been preserved and counted into all sessions?

## Late activity within allowed lateness (Fired Panes Accumulated)

To accumulate activities from already fired panes the accumulation mode has to be defined.

```scala
"Late activity within allowed lateness" should "be aggregated and accumulated into late pane" in runWithContext { sc =>
  val activities = testStreamOf[(User, Activity)]
    .addElementsAtTime("00:00:00", ("joe", "open app"))
    .addElementsAtTime("00:01:30", ("joe", "show product"))
    .advanceWatermarkTo("00:13:00")
    .addElementsAtTime("00:03:00", ("joe", "add to cart")) // late event within allowed lateness
    .addElementsAtTime("00:09:30", ("joe", "checkout"))
    .addElementsAtTime("00:13:10", ("joe", "close app"))
    .advanceWatermarkToInfinity()

  val results = activitiesInSessionWindow(
    sc.testStream(activities),
    TenMinutesGap,
    allowedLateness = Duration.standardMinutes(5),
    accumulationMode = AccumulationMode.ACCUMULATING_FIRED_PANES)

  results.withTimestamp should inOnTimePane("00:00:00", "00:11:30") {
    containSingleValueAtTime("00:11:29.999", ("joe", Iterable("open app", "show product")))
  }

  results.withTimestamp should inLatePane("00:00:00", "00:13:00") {
    containSingleValueAtTime("00:12:59.999", ("joe", Iterable("open app", "show product", "add to cart")))
  }

  results.withTimestamp should inOnTimePane("00:00:00", "00:23:10") {
    containSingleValueAtTime("00:23:09.999", ("joe", Iterable("open app", "show product", "add to cart", "checkout", "close app")))
  }
}
```

* The first session is produced as before
* The session in the late pane contains all activities from already fired, on time pane
* Now it is much easier to understand why late pane window boundaries are *[00:00:00, 00:13:00)*
* There is also second on time pane with all the events, the late event has closed the gap and the very long session is produced effectively

The aggregation with `ACCUMULATING_FIRED_PANES` mode could effectively produce some duplicates.
All downstream processing steps must be aware of that fact. 
In the above scenario they could use the latest observed session as the most completed.

## Speculative results

The last example in this blog post.
If the user plays with our e-commerce site for the very long time, the session would be produced at the very end of his/her journey.
It could take hours, what if we need more up-to-date results even if the session is not finished yet?

The following trigger defines that the aggregation should produce early, speculative results every minute after the first element has been seen.
When a watermark has passed the late results should be produced as well, on every observed late activity.

```scala
AfterWatermark
  .pastEndOfWindow()
  .withEarlyFirings(
    AfterProcessingTime.pastFirstElementInPane().plusDelayOf(OneMinute))
  .withLateFirings(
    AfterPane.elementCountAtLeast(1))
```

The complete example is presented below. 
In addition to the watermark advancement the processing time has to be advanced as well to fire early speculative results.
Instead of e-commerce action names I decided to introduce simple indices: 0, 1, 2 ... 
It should make the reasoning about processing time a little easier :)

```scala
"Activities" should "be aggregated speculatively on every minute, on-time, and finally on every late activity" in runWithContext { sc =>
  val OneMinute = Duration.standardMinutes(1L)

  val activities = testStreamOf[(User, Activity)]
    .addElementsAtTime("00:00:00", ("joe", "0")).advanceProcessingTime(OneMinute)
    .addElementsAtTime("00:01:00", ("joe", "1")).advanceProcessingTime(OneMinute)
    .addElementsAtTime("00:02:00", ("joe", "2")).advanceProcessingTime(OneMinute)
    .addElementsAtTime("00:03:00", ("joe", "3")).advanceProcessingTime(OneMinute)
    .addElementsAtTime("00:04:00", ("joe", "4")).advanceProcessingTime(OneMinute)
    .addElementsAtTime("00:05:00", ("joe", "5")).advanceProcessingTime(OneMinute)
    .advanceWatermarkTo("00:20:00") // more than 00:05:00 + 10 minutes of gap
    .addElementsAtTime("00:06:00", ("joe", "6")) // late event within allowed lateness
    .addElementsAtTime("00:07:00", ("joe", "7")) // late event within allowed lateness
    .advanceWatermarkToInfinity()

  val results = activitiesInSessionWindow(
    sc.testStream(activities),
    TenMinutesGap,
    accumulationMode = AccumulationMode.ACCUMULATING_FIRED_PANES,
    allowedLateness = Duration.standardMinutes(10),
    trigger = AfterWatermark
      .pastEndOfWindow()
      .withEarlyFirings(
        AfterProcessingTime.pastFirstElementInPane().plusDelayOf(OneMinute))
      .withLateFirings(
        AfterPane.elementCountAtLeast(1))
  )

  results.withTimestamp should inEarlyPane("00:00:00", "00:11:00") {
    containSingleValueAtTime("00:10:59.999", ("joe", Iterable("0", "1")))
  }
  results.withTimestamp should inEarlyPane("00:00:00", "00:13:00") {
    containSingleValueAtTime("00:12:59.999", ("joe", Iterable("0", "1", "2", "3")))
  }
  results.withTimestamp should inEarlyPane("00:00:00", "00:15:00") {
    containSingleValueAtTime("00:14:59.999", ("joe", Iterable("0", "1", "2", "3", "4", "5")))
  }
  results.withTimestamp should inOnTimePane("00:00:00", "00:15:00") {
    containSingleValueAtTime("00:14:59.999", ("joe", Iterable("0", "1", "2", "3", "4", "5")))
  }
  results.withTimestamp should inLatePane("00:00:00", "00:16:00") {
    containSingleValueAtTime("00:15:59.999", ("joe", Iterable("0", "1", "2", "3", "4", "5", "6")))
  }
  results.withTimestamp should inLatePane("00:00:00", "00:17:00") {
    containSingleValueAtTime("00:16:59.999", ("joe", Iterable("0", "1", "2", "3", "4", "5", "6", "7")))
  }
}
```

* To make a speculative results meaningful the `ACCUMULATING_FIRED_PANES` mode is configured
* The first speculative result is produced one minute after "0" activity, so it contains "0" and "1" activities
* The early pane trigger resets, so the second speculative result is produced 1 minute after "2" activity and contains activities from "0" to "3"
* The on-time pane is produced when watermark has passed more than the last observed activity plus the gap
* The late panes are triggered on every late element and the session contains all already observed actions
* You should already know why all the panes get the *00:00:00* as a start of the window, if not - study the previous examples again :)

## Summary

Feel free to inspect [source code](https://github.com/mkuthan/stream-processing) to get the whole picture of the examples.
Below you will find the actual `activitiesInSessionWindow` implementation:

```scala
type User = String
type Activity = String

def activitiesInSessionWindow(
  activities: SCollection[(User, Activity)],
  gapDuration: Duration,
  allowedLateness: Duration = Duration.ZERO,
  accumulationMode: AccumulationMode = AccumulationMode.DISCARDING_FIRED_PANES,
  trigger: Trigger = DefaultTrigger.of()
): SCollection[(User, Iterable[Activity])] = {
  val windowOptions = WindowOptions(
    allowedLateness = allowedLateness,
    accumulationMode = accumulationMode,
    trigger = trigger,
  )
  
  activities
    .withTimestamp
    .map { case ((user, action), timestamp) => (user, (action, timestamp)) }
    .withSessionWindows(gapDuration, windowOptions)
    .groupByKey
    .mapValues(sortByTimestamp)
    .mapValues(withoutTimestamp)
}

private def sortByTimestamp(activities: Iterable[(Activity, Instant)]): Iterable[(Activity, Instant)] = {
  val ordering: Ordering[(Activity, Instant)] = Ordering.by { case (_, instant) => instant.getMillis }
  actions.toSeq.sorted(ordering)
}

private def withoutTimestamp(activities: Iterable[(Activity, Instant)]): Iterable[Activity] =
  actions.map(_._1)
```

Key takeaways:

* Data-driven windows like session window widen the streaming processing patterns
* Session window is much more complex for reasoning and test than fixed window
* Late event under allowed lateness closes the gap of already produced sessions
* Session window could never end, but to get the speculative, early results you can define the special trigger

Let me know what you think as a comment below.
