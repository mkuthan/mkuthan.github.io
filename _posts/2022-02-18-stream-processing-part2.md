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

One of the most popular dynamic aggregations is done in session window. 
In this case single session captures user activities within some period of time followed by the gap of inactivity.

As an example, the following stream of activities is ingested from hypothetical mobile e-commerce application:

```
"00:00:00" -> "open app"
"00:01:30" -> "show product"
"00:03:00" -> "add to cart"
"00:13:00" -> "checkout"
"00:13:10" -> "close app"
```

The activities should be aggregated into single session if the maximum allowed gap between activities is equal or greater than 10 minutes:

```
00:23:10 -> "open app", "show product", "add to cart", "checkout", "close app"
```

Or into two independent sessions, if the allowed gap between activities is shorter, for 5 minutes gap it could be:

```
00:08:00 -> "open app", "show product", "add to cart"
00:18:10 -> "checkout", "close app"
```

This is over-simplified aggregation example just for this blog academic purposes, it does not scale at all. 
All user activities during the session must fit into memory of single machine.
In real-world scenario the actions should be reduced into scalable [algebraic](https://en.wikipedia.org/wiki/Algebraic_structure) data structures, e.g: 

* session of funnel length
* number of unique visited products
* number of deals
* click-through rate  
* conversion rate

I'm going to follow TDD technique again, so the implementation of the aggregation method will not be disclosed until the very end of this blog post.
As for now, the following method signature should be enough to understand all presented test scenarios:

```scala
type User = String
type Activity = String

def activitiesInSessionWindow(
  userActions: SCollection[(User, Activity)],
  gapDuration: Duration,
): SCollection[(User, Iterable[Activity])]
```

## No activities, no sessions

## Single session

## Two simultaneous sessions

## Single session with out-of-order activities

## Continues long-lasting session

## Interrupted long-lasting session

## Late activity

## Late activity within allowed lateness (Fired Panes Discarded)

## Late activity within allowed lateness (Fired Panes Accumulated)

## Speculative results

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

* TODO

Let me know what you think as a comment below.
