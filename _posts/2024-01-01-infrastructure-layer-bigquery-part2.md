---
title: "Infrastructure layer for unified batch and streaming data pipelines -- part 2"
date: 2023-01-01
tags: [Apache Beam, Scala, BigQuery, GCP]
---

TODO: refer to the part 1

## Design patters for implementing infrastructure layer

Before I present my design of the infrastructure layer API, I would like to show you Algebraic Data Types (ADT) and Phantom Types design patterns first.

### Algebraic Data Types

ADT is a concept in computer science and programming languages that describe data types by combining other data types in various ways.
There are two main types of ADTs:

**Sum Types (Disjoint Union)**: Sum types allow you to create a data type that can have one of several possible values. This is similar to a union or a tagged union in some other languages. The values in a sum type are mutually exclusive.
For example a simple sum type to define selected fields for Storage Read API.
You can select either list of fields or `NoFields`.

```scala
sealed trait SelectedFields

object SelectedFields {
  case object NoFields extends SelectedFields
  final case class NamedFields(fields: List[String]) extends SelectedFields
}
```

Similar sum type for Storage Read API row restriction, you can either define restriction as SQL or `NoRestriction`.

```scala
sealed trait RowRestriction

object RowRestriction {
  case object NoRestriction extends RowRestriction
  final case class SqlRestriction(sql: String) extends RowRestriction
}
```

**Product Types**: Product types allow you to combine multiple values into a single value, similar to a struct or record in some languages.
A product type can hold several values of different data types.
For example a simple product type to define Storage Read API configuration:

```scala
case class StorageReadConfiguration(
  rowRestriction: RowRestriction,
  selectedFields: SelectedFields
)
```

### Phantom types

Yet another useful design pattern for implementing infrastructure layer API are phantom types.
They help to catch type-related errors at compile-time, providing strong guarantees about the correctness of the code.
For example instead of using `String` to reference BigQuery tables define the following type:

```scala
case class BigQueryTable[T](id: String)

val tollBoothEntryTable = BigQueryTable[TollBoothEntry]("sample.toll_booth_entries")
val tollBoothExitTable = BigQueryTable[TollBoothExit]("sample.toll_booth_exits")
```

In this example, `BigQueryTable` is a parametrized type, and the type parameter `T` is used solely as a marker to distinguish different instances of `BigQueryTable` at the type level.
There is no runtime difference between `tollBoothEntryTable` and `tollBoothExitTable`, but the type parameter helps in making sure that the correct types are used in the right places and provides additional compile-time safety.
With such design the compiler complains if you try to read `TollBoothExit` records from `sample.toll_booth_entries` table.

## Opinionated infrastructure layer API

Armed with [#algebraic-data-types](Algebraic Data Types), [#phantom-types](Phantom Types) and knowing Apache Beam ans Spotify Scio disadvantages, let's design better API for infrastructure layer.
Deliver the following BigQuery connector features keeping the non-functional requirements in mind:

* Query using BigQuery jobs to fetch data with complex SQL queries.
The query jobs are extremely powerful with complex joins, aggregations but limited in testing.
* Read the table using Storage Read API if simple filtering and projection capabilities are enough.
I always prefer this API over SQL queries as it's easier to test.
* Write-truncate bounded collections of data into given partition using Batch Loads API.
This writing method optimizes for throughput required by batch pipelines.
* Write-append unbounded collections of data into given table using Storage Write API
This writing method optimizes for latency required by streaming pipelines.

Look at [Test Driven Development for Data Engineers](/blog/2023/11/02/tdd-for-de/) blog post to see how to test data pipelines with such infrastructure layer API.
{: .notice--info}

### SQL Query

The method takes parametrized `id`, `query` and returns collection of BigQuery records de-serialized as domain objects of type `T`.

* `IoIdenfitier` enables automated testing, you can stub the input using the identifier
* `BigQueryQuery` is a parametrized value class for SQL query
* `ExportConfiguration` is a configuration crafted only for Beam [BigQueryIO.TypedRead.Method.html#EXPORT](https://beam.apache.org/releases/javadoc/current/org/apache/beam/sdk/io/gcp/bigquery/BigQueryIO.TypedRead.Method.html#EXPORT).

```scala
def queryFromBigQuery[T](
  id: IoIdentifier[T],
  query: BigQueryQuery[T],
  configuration: ExportConfiguration = ExportConfiguration() // recommended defaults
): SCollection[T]
```

Example usage, look that `ioIdentifier`, `query` and `results` types match together.

```scala
val effectiveDate = LocalDate.parse(args.required("effectiveDate"))

val ioIdentifier = IoIdentifier[TollBoothEntry.Record]("toll.booth_entry")
val query = BigQueryQuery[TollBoothEntry.Record](
  s"SELECT * FROM toll.booth_entry WHERE _PARTITION_TIME = $effectiveDate"
)

val results = sc.queryFromBigQuery(ioIdentifier, query)
```

### Read

API:

```scala
def readFromBigQuery[T](
  id: IoIdentifier[T],
  table: BigQueryTable[T],
  configuration: StorageReadConfiguration = StorageReadConfiguration() // recommended defaults
): SCollection[T]
```

Example:

```scala
val effectiveDate = LocalDate.parse(args.required("effectiveDate"))

val ioIdentifier = IoIdentifier[TollBoothEntry.Record]("toll.booth_entry")
val table = BigQueryTable[TollBoothEntry.Record](args.required("entryTable")),
val rowRestriction = RowRestriction.SqlRestriction(s"_PARTITION_TIME = $effectiveDate")

val results = sc.readFromBigQuery(
  ioIdentifier,
  table,
  StorageReadConfiguration().withRowRestriction(rowRestriction)
)
```

### Write-truncate bounded collections

API:

```scala
def writeBoundedToBigQuery(
  id: IoIdentifier[T],
  partition: BigQueryPartition[T],
  configuration: FileLoadsConfiguration = FileLoadsConfiguration() // recommended defaults
): Unit
```

Example:

```scala
val effectiveDate = LocalDate.parse(args.required("effectiveDate"))

val ioIdentifier = IoIdentifier[TollBoothStats.Record]("toll.booth_stats_daily")
val partition = BigQueryPartition
  .daily[TollBoothStats.Record]("toll.booth_stats_daily", localDate)

val tollBoothStats: SCollection[TollBoothStats.Record] = ...

tollBoothEntries.writeBoundedToBigQuery(
  ioIdentifier,
  partition
)
```

### Write-append unbounded collections

API:

```scala
def writeUnboundedToBigQuery(
    id: IoIdentifier[T],
    table: BigQueryTable[T],
    configuration: StorageWriteConfiguration = StorageWriteConfiguration() // recommended defaults
): SCollection[BigQueryDeadLetter[T]]
```

Example:

```scala
val ioIdentifier = IoIdentifier[TollBoothStats.Record]("toll.booth_stats_realtime")
val table = BigQueryTable[TollBoothStats.Record]("toll.booth_stats_realtime")

val tollBoothStats: SCollection[TollBoothStats.Record] = ...
tollBoothEntries.writeUnboundedToBigQuery(
  ioIdentifier,
  table
)
```

## Summary

The most generic and reusable API isn't always the best choice.