# BigQuery partitioning - by time-unit column or by ingestion time

## Introduction

What's the best way to partition time-series data in BigQuery?
By time-unit column or by ingestion time? Daily or hourly?
It's depends, keep reading to learn trade-offs, pitfalls, and other traps.

## Time-series or aggregates

For time-series with data points with minute, second or millisecond precision use time-unit column partitioning.
You will get convenient querying like:

```sql
SELECT ts, temperate, pressure FROM weather_stations
WHERE ts BETWEEN TIMESTAMP("2008-12-25 15:30:00") AND TIMESTAMP("2008-12-25 15:35:00")
```

With ingestion time partitioning, you have to specify extra predicate for partition pruning:

```sql
SELECT ts, temperate, pressure FROM weather_stations
WHERE ts BETWEEN TIMESTAMP("2008-12-25 15:30:00") AND TIMESTAMP("2008-12-25 15:35:00")
AND _PARTITIONTIME = DATE("2008-12-15")
```

For aggregates, when you typically don't need exact point in time, use ingestion time partitioning.
You don't need to specify time column explicitly in a table schema.

```sql
SELECT _PARTITIONTIME as dt, page_views, unique_visitors FROM ecommerce_sessions
WHERE _PARTITIONTIME = DATE("2008-12-15")
```

## Retention

In my projects, majority of tables require 1--3 years of history.
With limit of 4000 partitions per BigQuery table, it requires at least daily partitioning.
Tables with 3 years of retention use `3 * 365 = 1095` daily partitions, below limit.
Tables with hourly partitions keep up to only `4000 / 24 = 166 days and 8 hours` of data.
For tables with more than 10 years of history I would consider monthly partitioning.

GCP support could raise the limit, for example to 10000 partitions but don't expect any guarantees in case of incidents
{.note}

## Timezones

If you need querying data using different timezones, use timestamp column partitioning.
The following query automatically reads data from two daily partitions: `2008-12-24 00:00:00Z` and `2008-12-25 00:00:00Z`.

```sql
SELECT temperate, pressure FROM weather_stations
WHERE ts BETWEEN TIMESTAMP("2008-12-25 00:00:00", "CET") AND TIMESTAMP("2008-12-26 00:00:00", "CET")
```

For ingestion time partitioning you could load data using table decorator and use whatever timezone you want instead of UTC.
If you load one day of data for "CET" timezone using `ecommerce_sessions$20081215` table decorator, the following query returns correct results:

```sql
SELECT page_views, unique_visitors FROM ecommerce_sessions
WHERE _PARTITIONTIME = DATE("2008-12-15")
```

Be aware, that you can't query for a range in another timezone than used while loading partition.
Moreover BigQuery always shows that `_PARTITIONTIME` uses UTC timezone, which will be misleading for poor users.

## Storage Write API

## Streaming Inserts

## Batch Loads

## Partition pruning

## Summary matrix
