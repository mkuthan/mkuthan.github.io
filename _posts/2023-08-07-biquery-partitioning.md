---
title: "BigQuery partitioning - by time-unit column or by ingestion time"
date: 2023-08-07
tags: [GCP, BigQuery]
header:
    overlay_image: /assets/images/2022-03-24-gcp-dataproc-spark-tuning/nana-smirnova-IEiAmhXehwE-unsplash.webp
    caption: "[Unsplash](https://unsplash.com/@nanasmirnova)"
---

What's the best way to partition time-series data in BigQuery?
By time-unit column or by ingestion time? Daily or hourly?
It depends, keep reading to learn trade-offs, pitfalls, and other traps.

## Time-series or aggregates

For time-series with data points with minute, second or millisecond precision use time-unit column partitioning.
You will get convenient querying like:

```sql
SELECT ts, temperate, pressure
FROM weather_stations
WHERE
    ts BETWEEN TIMESTAMP("2008-12-25 15:30:00")
        AND TIMESTAMP("2008-12-25 15:35:00")
```

With ingestion time partitioning, you have to specify extra predicate for partition pruning:

```sql
SELECT ts, temperate, pressure
FROM weather_stations
WHERE
    ts BETWEEN TIMESTAMP("2008-12-25 15:30:00")
        AND TIMESTAMP("2008-12-25 15:35:00")
    AND _PARTITIONTIME = DATE("2008-12-15")
```

For aggregates, when you typically don't need an exact point in time, you can use ingestion time partitioning.
You don't need to specify the time column explicitly in a table schema.

```sql
SELECT _PARTITIONTIME AS dt, page_views, unique_visitors
FROM ecommerce_sessions
WHERE
    _PARTITIONTIME = DATE("2008-12-15")
```

## Retention

In my projects, the majority of tables require at least 1--3 years of history.
With a limit of 4000 partitions per BigQuery table, it requires at least daily partitioning.
Tables with 3 years of retention use `3 * 365 = 1095` daily partitions, below limit.
Tables with hourly partitions keep up to only `4000 / 24 = 166 days and 8 hours` of data.

For tables with more than 10 years of history I would consider another storage than BigQuery.
[Export](https://cloud.google.com/bigquery/docs/exporting-data) partitions with data older than 10 years
to [Google Cloud Storage](https://cloud.google.com/storage/docs) bucket using columnar format like
[Apache Parquet](https://parquet.apache.org/docs/).
Create BigQuery [external table](https://cloud.google.com/bigquery/docs/external-data-cloud-storage) on the bucket
using Hive partitioning scheme for querying.

Google Cloud Platform support could raise the limit, for example to 10000 partitions but don't expect any guarantees in case of incidents
{: .notice--info}

## Timezones

If you need querying data using different timezones, use timestamp column partitioning.
The following query automatically reads data from two daily partitions: `2008-12-24 00:00:00Z` and `2008-12-25 00:00:00Z`.

```sql
SELECT temperate, pressure
FROM weather_stations
WHERE
    ts BETWEEN TIMESTAMP("2008-12-25 00:00:00", "CET")
        AND TIMESTAMP("2008-12-26 00:00:00", "CET")
```

For ingestion time partitioning you could load data using table decorator and use whatever timezone you want instead of UTC.
If you load one day of data for Central European Time (CET) timezone using `ecommerce_sessions$20081215` table decorator, the following query returns correct results:

```sql
SELECT DATE(_PARTITIONTIME) AS dt, page_views, unique_visitors
FROM ecommerce_sessions
WHERE
    _PARTITIONTIME = DATE("2008-12-15")
```

Be aware, that you can't query for a range in another timezone than used while loading partitions.
Moreover BigQuery always shows that `_PARTITIONTIME` uses UTC timezone, which will be misleading for users.

If you don't need [half-hour or 45-minutes](https://www.timeanddate.com/time/time-zones-interesting.html) timezones,
and long retention, you could use hourly partitioned tables by ingestion time using UTC.

The following query reads 24 hourly partitions and returns correct results:

```sql
SELECT TIMESTAMP(_PARTITIONTIME, "CET") AS ts, page_views, unique_visitors
FROM ecommerce_sessions
WHERE
    _PARTITIONTIME = BETWEEN TIMESTAMP("2008-12-25 00:00:00", "CET")
        AND TIMESTAMP("2008-12-26 00:00:00", "CET")
```

## Storage Write API

If you want to use [Storage Write API](https://cloud.google.com/bigquery/docs/write-api)
for partitioned tables use column partitioning.
The Storage Write API doesn't support the use of partition decorators to write to the given partition.

## Streaming Inserts

If you want to use [Streaming Inserts](https://cloud.google.com/bigquery/docs/streaming-data-into-bigquery)
for partitioned tables use column partitioning.
The Streaming Inserts has limited support for partition decorators.
You can stream to partitions within the last 31 days in the past and 16 days in the future relative to the current date,
based on current UTC time.

## Streaming buffer

The Storage Write API and Streaming Inserts write data through the streaming buffer.
For ingestion time partitioned tables data in streaming buffer is temporary placed in the `__UNPARTITIONED__` partition and has a `NULL` value in `_PARTITIONTIME` column.
One more reason to not use ingestion time partitioned tables for Storage Write API or Streaming Inserts.
Querying such tables is error prone.

## Batch Loads

I'm not aware of any [Batch Loads](https://cloud.google.com/bigquery/docs/load-data-partitioned-tables)
limitations for partitioned tables.

## Partition pruning

If you process data on a daily basis use daily partitioning for efficient partition pruning.
If you process data on an hourly basis and don't need 6+ months of history in the table, use hourly partitioning.

If you need to keep longer history use daily partitioning and one of the following tricks for efficient querying:

1. For timestamp-column partitioning define also a clustering on the partitioning column.
2. For ingestion time partitioning add an "hour" or "minute of the day" column and define clustering on this column.

For the trick with clustering on timestamp partitioning column the following query reads only 1 minute of data in daily partitioned table:

```sql
SELECT ts, temperate, pressure
FROM weather_stations
WHERE
    ts BETWEEN TIMESTAMP("2008-12-25 15:30:00")
        AND TIMESTAMP("2008-12-25 15:31:00")
```

However, the timestamp clustering column has a huge entropy, so if you need more clustering columns you can't use this trick.
{: .notice--info}

For the trick with extra "hour" clustering column the following query reads one hour of data in daily partitioned table:

```sql
SELECT ts, temperate, pressure
FROM weather_stations
WHERE
    ts BETWEEN TIMESTAMP("2008-12-25 15:30:00")
        AND TIMESTAMP("2008-12-25 15:31:00")
    AND _PARTITIONTIME = DATE("2008-12-15")
    AND hour = 15
```

As you see, such a table isn't convenient to query, the client must be aware of two extra predicates.

## Summary matrix

Below you can find the matrix with cons and pros of different partitioning methods in BigQuery.

| | time-unit column daily | time-unit column hourly | ingestion time daily | ingestion time hourly |
| --- | --- | --- | --- | --- |
| Best for time-series | ✅ | ✅ | 🚫 | 🚫 |
| Best for aggregates | 🚫 | 🚫 | ✅ | ✅ |
| 6+ months of retention | ✅ | 🚫 | ✅ | 🚫 |
| 10+ years of retention | 🚫 | 🚫 | 🚫 | 🚫 |
| UTC only timezone | ✅ | ✅ | ✅ | ✅ |
| Non-UTC timezone | ✅ | ✅ | limited | limited |
| Many timezones | ✅ | ✅ | 🚫 | limited |
| Storage Write API | ✅ | ✅ | 🚫 | 🚫 |
| Streaming Inserts | ✅ | ✅ | limited | limited |
| Batch Loads | ✅ | ✅ | ✅ | ✅ |
| Partition pruning | natural | natural | less convenient | less convenient |

I would always prefer time-unit column partitioning with daily granularity as the least problematic.
