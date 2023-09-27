---
title: "Apache Beam testing"
date: 2023-10-31
tags: [Apache Beam, Scala, Software Engineering]
header:
    overlay_image: /assets/images/2015-03-01-spark-unit-testing/jakub-skafiriak-AljDaiCbCVY-unsplash.webp
    caption: "[Unsplash](https://unsplash.com/@jakubskafiriak)"
---

In 2015 I blogged how to test [Spark and Spark Streaming](/blog/2015/03/01/spark-unit-testing/) data pipelines.
Today, I'm not fully agree with myself from the past because the context has changed, the tools are more mature and data pipelines are more complex.
However, I still think that capabilities for automated testing are one of the most important element of modern data processing frameworks.

This blog post gathers many testing practices I use on daily basis for developing [unified batch and streaming](/blog/2023/09/27/unified-batch-streaming/) data pipelines:

* Codebase organization for testability
* Business logic tests on local runner
* How to prepare test data in programmatic and reusable way?
* How to test out-of-order or late data?
* When to use property-based testing?
* Tests for the whole pipelines with stubbed sources and sinks
* Integration tests for cloud based services like BigQuery, Pubsub or Cloud Storage

All presented code examples use [Apache Beam](https://github.com/apache/beam) and [Spotify Scio](https://github.com/spotify/scio), but general rules are portable to other data processing framework like [Apache Flink](https://flink.apache.org) or [Apache Spark](https://flink.apache.org).
Or use Apache Beam / Spotify Scio API to develop data pipelines and run them on Flink, Spark or any other supported [runner](https://beam.apache.org/documentation/runners/capability-matrix/).
