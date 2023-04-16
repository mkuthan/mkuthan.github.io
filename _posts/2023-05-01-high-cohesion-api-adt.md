---
title: "High cohesion API design with algebraic data type"
date: 2023-04-03
tags: [Software Engineering, Scala, Apache Beam]
header:
    overlay_image: /assets/images/2023-05-01-high-cohesion-api-adt/mourizal-zativa-OSvN1fBcXYE-unsplash.webp
    caption: "[Unsplash](https://unsplash.com/@mourimoto)"
---

API design is one of the more demanding software engineering field.
The API should be able to evolve and clients should discover all functionalities without reading a documentation.
For strongly typed languages like Scala, all faulty API usages should generate compile time errors.

In this blog post I would focus on [cohesion](https://en.wikipedia.org/wiki/Cohesion_%2Fcomputer_science),
or rather lack of cohesion in [Apache Beam](https://beam.apache.org/documentation/io/built-in/google-bigquery/) BigQuery I/O connector.
Instead, you will find elegant, strongly typed adapter for the Apache Beam API implemented with [algebraic data types](https://en.wikipedia.org/wiki/Algebraic_data_type).

## Apache Beam BigQuery connector

All Apache Beam built-in connectors use [builder pattern](https://en.wikipedia.org/wiki/Builder_pattern) to configure I/O.
It's an object oriented design pattern for creating complex objects step by step.
For example, to write to BigQuery, you need to create a `BigQueryIO.Wite` object and configure it with a table name, writing method, number of shards and create/write dispositions.

```java
BigQueryIO.Write write = BigQueryIO.write()
    .to("project:dataset.table")
    .withMethod(Method.FILE_LOADS)
    .withNumFileShards(3)
    .withCreateDisposition(CreateDisposition.CREATE_IF_NEEDED)
    .withWriteDisposition(WriteDisposition.WRITE_APPEND)
```

Builder pattern might be a good solution for configuring simple I/O connectors
but for BigQuery connector it's a nightmare.
Below is the list of methods in `BigQueryIO.Write` builder class:

1. `ignoreInsertIds()`
1. `ignoreUnknownValues()`
1. `optimizedWrites()`
1. `skipInvalidRows()`
1. `useAvroLogicalTypes()`
1. `useBeamSchema()`
1. `withAutoSchemaUpdate(boolean autoSchemaUpdate)`
1. `withAutoSharding()`
1. `withAvroWriter(SerializableFunction<Schema,DatumWriter<T>> writerFactory)`
1. `withClustering(Clustering clustering)`
1. `withCreateDisposition(BigQueryIO.Write.CreateDisposition createDisposition)`
1. `withCustomGcsTempLocation(ValueProvider<java.lang.String> customGcsTempLocation)`
1. `withExtendedErrorInfo()`
1. `withFailedInsertRetryPolicy(InsertRetryPolicy retryPolicy)`
1. `withJsonSchema(java.lang.String jsonSchema)`
1. `withKmsKey(java.lang.String kmsKey)`
1. `withLoadJobProjectId(java.lang.String loadJobProjectId)`
1. `withMaxBytesPerPartition(long maxBytesPerPartition)`
1. `withMaxFilesPerBundle(int maxFilesPerBundle)`
1. `withMaxRetryJobs(int maxRetryJobs)`
1. `withMethod(BigQueryIO.Write.Method method)`
1. `withNumFileShards(int numFileShards)`
1. `withNumStorageWriteApiStreams(int numStorageWriteApiStreams)`
1. `withoutValidation()`
1. `withSchema(TableSchema schema)`
1. `withSchemaUpdateOptions(java.util.Set<BigQueryIO.Write.SchemaUpdateOption> schemaUpdateOptions)`
1. `withSuccessfulInsertsPropagation(boolean propagateSuccessful)`
1. `withTableDescription(java.lang.String tableDescription)`
1. `withTimePartitioning(TimePartitioning partitioning)`
1. `withTriggeringFrequency(Duration triggeringFrequency)`
1. `withWriteDisposition(BigQueryIO.Write.WriteDisposition writeDisposition)`
1. `withWriteTempDataset(java.lang.String writeTempDataset)`

Why there are so many knobs?
Because the connector supports many different BigQuery APIs:

* [BigQuery Query API](https://cloud.google.com/bigquery/docs/reference/rest/v2/jobs/query) for reading
* [BigQuery Storage API](https://cloud.google.com/bigquery/docs/reference/storage/) for writing and reading
* [BigQuery Load API](https://cloud.google.com/bigquery/docs/reference/rest/v2/jobs/insert) for writing
* [BigQuery Insert API](https://cloud.google.com/bigquery/docs/reference/rest/v2/tabledata/insertAll) for writing

Some configuration options are relevant for single API, for example: `withAutoSchemaUpdate(boolean autoSchemaUpdate)` or `withNumStorageWriteApiStreams(int numStorageWriteApiStreams)` are valid for `BigQuery Storage API`.
Some configurations are heavily dependant, for example when writing an unbounded `PCollection` via `FILE_LOADS` method you need to specify `withTriggeringFrequency` and `withNumFileShards`.
Some combinations don't make any sense, for example `withCreateDisposition(CREATE_NEVER)` and `withClustering` or `withTimePartitioning`.
You can easily make a mistake and configure the connector incorrectly.
The only way to find out is to run the pipeline and wait for the error.

## Algebraic data types

Algebraic data types (ADT) are a way to model data in a type safe way.
Two common types of ADTs are product types and sum types.

A product type is a type that combines two or more types into a new type.
The resulting type has all possible combinations of the constituent types.

In Scala, a case class represents a product type.

Product type for BigQuery Storage Write API configuration:

```scala
case class StorageWriteConfiguration(
    createDisposition: CreateDisposition,
    writeDisposition: WriteDisposition
    method: StorageWriteMethod,
)
```

Product type for BigQuery Storage Read API configuration:

```scala
case class StorageReadConfiguration(
    rowRestriction: RowRestriction,
    selectedFields: SelectedFields
)
```

A sum type is a type that represents a choice between two or more types.

Sum types for BigQuery Storage Write API:

```scala
sealed trait CreateDisposition

object CreateDisposition {
  case object CreateNever extends CreateDisposition
  case object CreateIfNeeded extends CreateDisposition
}
```

```scala
object WriteDisposition {
  case object WriteAppend extends WriteDisposition
  case object WriteEmpty extends WriteDisposition
  case object WriteTruncate extends WriteDisposition
}
```

```scala
sealed trait StorageWriteMethod

object StorageWriteMethod {
  case object ExactlyOnce extends StorageWriteMethod
  case object AtLeastOnce extends StorageWriteMethod
}
```

Sum types for BigQuery Storage Read API:

```scala
sealed trait RowRestriction

object RowRestriction {
  case object NoRowRestriction extends RowRestriction
  case class SqlRowRestriction(sql: String)
}
```

```scala
sealed trait SelectedFields extends StorageReadParam

object SelectedFields {
  case object NoSelectedFields extends SelectedFields
  case class NamedSelectedFields(fields: List[String])
}
```

Product and sum types of ADTs are commonly used in functional programming languages to give a powerful way of creating complex data structures.
