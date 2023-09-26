---
title: "Unified batch and streaming"
date: 2023-09-27
tags: [Stream Processing, Apache Beam, Scala, GCP]
header:
    overlay_image: /assets/images/2023-09-27-unified-batch-streaming/lukas-bato-6Au5QdBsGuc-unsplash.webp
    caption: "[Unsplash](https://unsplash.com/@lks_bt)"
---

Unified batch and streaming processing is a data architecture that seamlessly combines both batch and real-time data processing.
It enables organizations to gain real-time insights from their data while maintaining the ability to process large volumes of historical data. In the past organizations often dealt with batch and streaming as separate data pipelines,
leading to increased complexity in data infrastructure and codebase management.

## Lambda architecture

Below you can find the lambda architecture of the clickstream data enrichment system I developed around 2017. If you have never heard about lambda architecture, see Nathan Marz article [How to beat the CAP theorem](http://nathanmarz.com/blog/how-to-beat-the-cap-theorem.html).

```mermaid
graph LR
    subgraph speed-layer
      kafka1[Kafka]-->kafka-streams[Kafka Streams]
      kafka-streams-->kafka2[Kafka]
    end

    subgraph batch-layer
      hdfs1[HDFS]-->spark[Apache Spark]
      spark-->hdfs2[HDFS]
    end

    kafka2-->druid[Apache Druid]
    hdfs2-->druid

    lib(anemic domain library) .- kafka-streams
    lib .- spark
```

Real-time data pipeline (a.k.a speed layer) consumed an infinite stream of events from Kafka cluster, performed stateless or stateful transformations, and published results again to Kafka.
Batch data pipeline read complete partition of data from HDFS, performed transformations and wrote results to HDFS.
Analytical databases like Apache Druid loaded real-time and batch results and acted as a serving layer for downstream consumers.
To reduce duplication I extracted part of domain logic into a shared library.
Speed and batch layers used the library to apply common processing logic.

In practice, such system had the following design flaws:

* Kafka Streams and Apache Spark used different API for stateful operations like windowing and joining.
I couldn't reuse the code between speed and batch layers beyond the stateless map/filter operations.

* Runtime environments for speed and batch layer were different. Kafka Streams run on Mesos cluster, Apache Spark on YARN. I couldn't reuse deployment automation, monitoring and alerting.

* While Apache Spark was a mature data processing framework, Kafka Streams didn't meet my expectations. See my blog post [Kafka Streams DSL vs processor API](/blog/2017/11/02/kafka-streams-dsl-vs-processor-api/) from 2017 for details.

## Unified programming model

In 2019 I moved from on-prem Hadoop to Google Cloud Platform and changed technology stack for developing data pipelines to [Apache Beam](https://github.com/apache/beam) / [Spotify Scio](https://github.com/spotify/scio).
It's still a lambda architecture but realization is much better than in 2017.

```mermaid
graph LR
    subgraph Apache-Beam
      direction TB
      speed(speed layer).-lib(rich domain library).-batch(batch layer)
    end

    ps-in[Pubsub]-->Apache-Beam
    bq-in[BigQuery]-->Apache-Beam

    Apache-Beam-->ps-out[Pubsub]
    Apache-Beam-->bq-out[BigQuery]
```

If I compare new architecture to the architecture from 2017:

* Apache Beam allows to unify domain logic between batch and speed layers.
Stateful operations like window joins or temporal joins are the same for streaming and batch.

* Runtime environments for real-time and batch are almost the same.
I deploy all pipelines on Dataflow, managed service on Google Cloud Platform.

* Maturity of batch and streaming parts is similar.
For example, Dataflow provides external services for data shuffling in batch and streaming.

However, the unification doesn't mean that you can deploy exactly the same job in a streaming or in a batch manner.
There are no magic parameters like `--batch` or `--streaming`, you have to build such capability yourself by proper design.
{: .notice--info}

## Data pipeline

Let's start with a simple use-case where unified batch and streaming delivers real business value.
Data pipeline for calculating statistics from toll booths you encounter on highways, bridges, or tunnels.
I took inspiration from Azure documentation [Build an IoT solution by using Stream Analytics](https://learn.microsoft.com/en-us/azure/stream-analytics/stream-analytics-build-an-iot-solution-using-stream-analytics).

```mermaid
graph LR
    toll-booth-entry[Toll booth entries]-->pipeline[[Unified Data Pipeline]]
    toll-booth-exit[Toll booth exits]-->pipeline
    vehicle-registration-history[Vehicle registrations]-->pipeline

    pipeline-->toll-booth-stat[Toll booth statistics]
    pipeline-->total-vehicle-time[Total vehicle times]
    pipeline-->vehicles-with-expired-registration[Vehicles with expired registrations]

    pipeline-.->diagnostic[Diagnostics]
    pipeline-.->dlq[Dead letters]
```

### Sources

Streaming pipeline subscribes for events emitted when vehicles cross toll booths.
On startup it also reads the history of vehicle registrations and after start gets real-time updates. The command line could looks like this:

```
TollBoothStreamingJob \
    --entrySubscription=...
    --exitSubscription=...
    --vehicleRegistrationTable=...
    --vehicleRegistrationSubscription=...
```

Batch version of the same pipeline reads historical data from a data warehouse and
calculates results for a given date specified as `effeciveDate` parameter.

```
TollBoothBatchJob \
    --effectiveDate=2014-09-04
    --boothEntryTable=...
    --boothExitTabel=...
    --vehicleRegistrationTable=...
```

The sources of the pipelines are remarkably different.
Streaming expects streams of near real-time data to deliver low latency results.
Batch requires efficient and cheap access to historical data to process large volumes of data at once.

### Sinks

Regardless of the source of data, batch and streaming pipelines calculate similar statistics, for example:

* Count the number of vehicles that enter a toll booth.
* Report total time for each vehicle

The streaming job aggregates results in short windows to achieve low latency.
It publishes statistics as streams of events for downstream real-time data pipelines.
The streaming pipeline also detects vehicles with expired registrations, it's more valuable than fraud detection in a daily batch.

```
TollBoothStreamingJob \
    --entryStatsTopic=...
    --totalVehicleTimeTopic=...
    --vehiclesWithExpiredRegistrationTopic=...
```

The batch job aggregates statistics in much larger windows to achieve better accuracy.
It writes results into a data warehouse for downstream batch data pipelines and reporting purposes.

```
TollBoothBatchJob \
    --effectiveDate=2014-09-04
    --entryStatsTable=...
    --totalVehicleTimeTable=...
```

As you could see, the sinks of the pipelines are also different.
The streaming publishes low latency, append-only results as streams of events,
batch overwrites the whole partitions in data warehouse tables for specified `effectiveDate`.

### Diagnostics

Because the streaming pipeline is hard to debug, it's crucial to put aside some diagnostic information about the current state of the job.
For example if the job receives a toll booth entry message for a given vehicle, but it doesn't have information about this vehicle registration yet.
This is a temporary situation if one stream of data is late (vehicle registrations) and the job produces incomplete results.
With proper diagnostic you could decide to change streaming configuration and increase allowed lateness for the windowing calculation.

### Dead letters

If there is an error in the input data, the batch pipeline just fails.
You could fix invalid data and execute a batch job again.

In the streaming pipelines, a single invalid record could block the whole pipeline forever. How to handle such situations?

* Ignore such errors but you will lost the pipeline observability
* Log such errors but you can easily exceed logging quotas
* **Put invalid messages with errors into Dead Letter Queue (DLQ) for inspection and reprocessing**

## Data pipeline layers

As you could see, the streaming and batch pipelines aren't the same.
They have different sources and sinks, use different parameters to achieve either lower latency or better accuracy.

How to organize the code to get unified architecture?
{: .notice--info}

Split the codebase into three layers:

* Domain with business logic, shared between streaming and batch
* Infrastructure with sources and sinks (I/0)
* Application to parse command line parameters and glue Domain and Infrastructure together

```mermaid
graph TB
    application-. depends on .->domain
    application-. uses .->infrastructure
```

Direct dependency between Domain and Infrastructure is forbidden, it would kill code testability.

### Domain

Below you can find a function for calculating total time between vehicle entry and exit.
The function uses a session window to join vehicle entries and vehicle exits within a gap duration.
When there is no exit for a given entry, the function can't calculate total time but emits diagnostic information.

```scala
import org.joda.time.Duration
import com.spotify.scio.values.SCollection

def calculateInSessionWindow(
    boothEntries: SCollection[TollBoothEntry],
    boothExits: SCollection[TollBoothExit],
    gapDuration: Duration
): (SCollection[TotalVehicleTime], SCollection[TotalVehicleTimeDiagnostic]) = {
    val boothEntriesById = boothEntries
        .keyBy(entry => (entry.id, entry.licensePlate))
        .withSessionWindows(gapDuration)
    val boothExistsById = boothExits
        .keyBy(exit => (exit.id, exit.licensePlate))
        .withSessionWindows(gapDuration)

    val results = boothEntriesById
        .leftOuterJoin(boothExistsById)
        .values
        .map {
        case (boothEntry, Some(boothExit)) =>
            Right(toTotalVehicleTime(boothEntry, boothExit))
        case (boothEntry, None) =>
            Left(TotalVehicleTimeDiagnostic(boothEntry.id, TotalVehicleTimeDiagnostic.MissingTollBoothExit))
        }

    results.unzip
}

private def toTotalVehicleTime(boothEntry: TollBoothEntry, boothExit: TollBoothExit): TotalVehicleTime = {
    val diff = boothExit.exitTime.getMillis - boothEntry.entryTime.getMillis
    TotalVehicleTime(
        licensePlate = boothEntry.licensePlate,
        tollBoothId = boothEntry.id,
        entryTime = boothEntry.entryTime,
        exitTime = boothExit.exitTime,
        duration = Duration.millis(diff)
    )
}
```

The logic is exactly the same for the streaming and for the batch, there is no I/O related code here.
Streaming pipeline defines shorter window gap to get lower latency,
batch pipeline longer gap for better accuracy, this is the only difference.

Because this is a data processing code, don't expect a pure domain without external dependencies. Domain logic must depend on Apache Beam / Spotify Scio to make something useful.
{: .notice--info}

### Application

Application layer is a main differentiator between batch and streaming.

A slightly simplified version of the streaming pipeline might look like this:

```scala
def main(mainArgs: Array[String]): Unit = {
    val (sc, args) = ContextAndArgs(mainArgs)
    val config = TollStreamingJobConfig.parse(args)

    // subscribe to toll booth entries and exits as JSON, put invalid messages into DLQ
    val (entryMessages, entryMessagesDlq) =
      sc.subscribeJsonFromPubsub(config.entrySubscription)
    val (exitMessages, exitMessagesDlq) =
      sc.subscribeJsonFromPubsub(config.exitSubscription)

    // decode JSONs into domain objects
    val (entries, entriesDlq) = TollBoothEntry.decodeMessage(entryMessages)
    val (exits, existsDlq) = TollBoothExit.decodeMessage(exitMessages)

    // write invalid inputs to Cloud Storage
    entriesDlq
      .withFixedWindows(duration = TenMinutes)
      .writeUnboundedToStorageAsJson(config.entryDlq)
    existsDlq
      .withFixedWindows(duration = TenMinutes)
      .writeUnboundedToStorageAsJson(config.exitDlq)

    // calculate total vehicle times
    val (totalVehicleTimes, totalVehicleTimesDiagnostic) =
      TotalVehicleTime.calculateInSessionWindow(entries, exits, gapDuration = TenMinutes)

    // write aggregated diagnostic to BigQuery
    TotalVehicleTimeDiagnostic
      .aggregateAndEncode(totalVehicleTimesDiagnostic, windowDuration = TenMinutes)
      .writeUnboundedToBigQuery(config.totalVehicleTimeDiagnosticTable)

    // encode total vehicle times as a message and publish on Pubsub
    TotalVehicleTime
      .encodeMessage(totalVehicleTimes)
      .publishJsonToBigQuery(config.totalVehicleTimeTable)

    // encode total vehicle times and writes into BigQuery, put invalid writes into DLQ
    val totalVehicleTimesDlq = TotalVehicleTime
      .encodeRecord(totalVehicleTimes)
      .writeUnboundedToBigQuery(config.totalVehicleTimeTable)

    // union all DLQs as I/O diagnostics, aggregate and write to BigQuery
    val ioDiagnostics = IoDiagnostic.union(
      boothEntryMessagesDlq,
      boothExitMessagesDlq,
      totalVehicleTimesDlq
    )
    ioDiagnostics
      .sumByKeyInFixedWindow(windowDuration = TenMinutes)
      .writeUnboundedToBigQuery(config.diagnosticTable)

    // run the pipeline
    sc.run()
}
```

The corresponding batch pipeline is less complex and looks like this:

```scala
def main(mainArgs: Array[String]): Unit = {
    val (sc, args) = ContextAndArgs(mainArgs)

    val config = TollBatchJobConfig.parse(args)

    // read toll booth entries and toll booth exists from BigQuery partition
    val entryRecords = sc.readFromBigQuery(
        config.entryTable,
        StorageReadConfiguration().withRowRestriction(
            RowRestriction.TimestampColumnRestriction("entry_time", config.effectiveDate)
        )
    )
    val exitRecords = sc.readFromBigQuery(
      config.exitTable,
      StorageReadConfiguration().withRowRestriction(
        RowRestriction.TimestampColumnRestriction("exit_time", config.effectiveDate)
      )
    )

    // decode BigQuery like objects into domain objects
    val entries = TollBoothEntry.decodeRecord(entryRecords)
    val exits = TollBoothExit.decodeRecord(exitRecords)

    // calculate total vehicle times
    val (totalVehicleTimes, totalVehicleTimesDiagnostic) =
      TotalVehicleTime.calculateInSessionWindow(entries, exits, gapDuration = OneHour)

    // write aggregated diagnostic to BigQuery
    TotalVehicleTimeDiagnostic
      .aggregateAndEncode(totalVehicleTimesDiagnostic, windowDuration = OneDay)
      .writeBoundedToBigQuery(config.totalVehicleTimeDiagnosticOneHourGapTable)

    // encode total vehicle times and writes into BigQuery
    TotalVehicleTime
      .encodeRecord(totalVehicleTimes)
      .writeBoundedToBigQuery(config.totalVehicleTimeOneHourGapPartition)

    // run the pipeline
    sc.run()
}
```

Upon initial glance streaming and batch pipelines look like a duplicated code which violates DRY principle (Don't Repeat Yourself). Where's batch and streaming unification?

Don't worry, nothing is wrong with such design:

* Application layer should use Descriptive and Meaningful Phrases (DAMP principle) over DRY
* Configuration is different, inspect properties of `TollStreamingJobConfig` and `TollBatchJobConfig`
* Sources are different, Pubsub subscriptions for streaming and BigQuery tables for batch
* Results with total vehicle times aggregated within different session gaps, don't mix datasets of different accuracy in a single table
* Streaming performs dual writes to Pubsub and BigQuery
* Error handling for streaming is much more complex

The example application doesn't use dependency injection framework.
Trust me, you don't need any to write manageable data pipelines.
{: .notice--info}

### Infrastructure

Where's the infrastructure in the presented code samples?
Because of Scala syntax powered by some implicit conversions it's hard to spot.

Infrastructure layer provides all the functions specified below in a fully generic way.

* `subscribeJsonFromPubsub` - for subscribing to JSON messages on Pubsub
* `publishJsonToPubsub` - for publishing JSON messages on Pubsub
* `readFromBigQuery` - for reading from BigQuery using Storage Read API
* `queryFromBigQuery` - for querying BigQuery using SQL
* `writeUnboundedToBigQuery` - for writing to BigQuery using Storage Write API with append mode
* `writeBoundedToBigQuery` - for writing to BigQuery using File Loads with truncate mode
* `writeUnboundedToStorageAsJson` - for writing JSON files with dead letters on Cloud Storage

Extract the infrastructure module as a shared library and reuse it in all data pipelines.
Writing and testing I/O is complex so do it once, and do it well.

## Summary

Unified batch and streaming doesn't mean that you can execute exactly the same code in a batch or streaming fashion.
It doesn't work this way, streaming pipelines favor lower latency, batch is for higher accuracy.

* Organize your codebase into domain, application and infrastructure layers
* Keep business logic in domain layer and make it fully reusable between batch and streaming
* Invest into reusable infrastructure layer with high quality IO connectors
* Keep application layer descriptive and delegate complex tasks to domain and infrastructure layers
* Don't try to replace batch by streaming alone, streaming is complex and not as cost effective as batch
* Start with the batch pipeline, and then move gradually to the streaming one

I take all code samples from [https://github.com/mkuthan/stream-processing/](https://github.com/mkuthan/stream-processing/), my playground repository for stream processing.
If you find something interesting but foggy in this repository, just let me know.
I will be glad to blog something more about stream processing.
