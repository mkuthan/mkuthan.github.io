---
layout: post
title: "Apache BigData Europe Conference Summary"
date: 2016-11-18
comments: true
categories: [conferences]
---

Last week I attended [Apache Big Data Europe](http://events.linuxfoundation.org/events/apache-big-data-europe) held in Sevilla, Spain. 
The event concentrates around big data projects under [Apache Foundation](https://www.apache.org/) umbrella. 
Below you can find my overall impression on the conference and notes from several interesting sessions.
The notes are presented as a short checklists, if some aspect was particularly interesting I put the reference to supplementary materials. 

## Key takeaways

* At [Allegro](http://allegro.tech/) we are on track with our clickstream ingestion platform.
[Apache Kafka](https://kafka.apache.org/), [Apache Avro](https://avro.apache.org/), [Apache Parquet](https://parquet.apache.org/), [Apache Spark](https://spark.apache.org/), [Apache Hive](https://hive.apache.org/) and last but not least [Apache Druid](http://druid.io/) are key players for us, all hosted under [Apache Foundation](https://www.apache.org/)!
* [Apache Ignite](https://ignite.apache.org/) might solve many performance issues in Spark jobs (shared RDD) but also in MR jobs (in-memory MR, HDFS cache). 
Has to be verified during next Allegro internal hackaton, for sure.
* Integration between [Apache Hive](https://hive.apache.org/) and [Apache Druid](http://druid.io/) looks really promising. 
Both tools are very important for Hadoop ecosystem and they complement each other quite well.
* [Apache Calcite](https://calcite.apache.org/) seems to be important element in the Hadoop ecosystem.
I hope that the gap to mature RDBMS optimizers will be somehow filled. 
It would be also great to see [Spark Catalyst](http://people.csail.mit.edu/matei/papers/2015/sigmod_spark_sql.pdf) and [Apache Calcite](https://calcite.apache.org/) cooperation, keep your fingers crossed.
* Stephan Even should improve [Apache Flink](https://flink.apache.org/) keynotes if DataArtisans want to compete with DataBricks. 
[Apache Flink](https://flink.apache.org/) architecture and overall design is awesome, FTW.
* Queryable state in stream processing is quite interesting idea to decrease latency in access to pre-aggregated data. 
* [Apache Kudu](https://kudu.apache.org/) and [Apache Impala](https://impala.apache.org/) are on dead end, IMHO. 
The concept to execute analytical queries (fast SQL on Impala) against whole set of raw data (kept in Kudu) is unrealistic. 
Cloudera gains mastery in keeping their own technologies alive (e.g: [Apache Flume](https://flume.apache.org/)).
* [Apache Gearpump](https://gearpump.apache.org/) from Intel has lost its momentum. I really liked idea of distributed streaming framework built on Akka.
* I was really surprised that CSV format is heavily used and what is even worse, conference speakers still talk about it.

## TL;DR

### “Stream processing as a foundational paradigm and Apache Flink's approach to it” by Stephan Ewen (Data Artisans)

* With Flink you don’t have to trade off either latency, throughput, or result accuracy - nice, single sentence to describe the framework.
* Asynchronous Distributed Snapshot is a key to achieve fault tolerance and avoid “stop the world”. 
See also:
[https://ci.apache.org/projects/flink/flink-docs-master/internals/stream_checkpointing.html](https://ci.apache.org/projects/flink/flink-docs-master/internals/stream_checkpointing.html)
* Application rolling updates with state versioning.
See also: 
[http://data-artisans.com/how-apache-flink-enables-new-streaming-applications/](http://data-artisans.com/how-apache-flink-enables-new-streaming-applications/)
* Roadmap: [elastic parallelism](http://flink-forward.org/wp-content/uploads/2016/07/Till-Rohrmann-Dynamic-Scaling-How-Apache-Flink-adapts-to-changing-workloads.pdf) (to scale out/in stateful jobs),
realtime queries using [Apache Calcite Streaming SQL](https://calcite.apache.org/docs/stream.html).

### “Apache Gearpump next-gen streaming engine” by Karol Brejna, Huafeng Wang (Intel)

* [Trusted Analytics Platform](http://trustedanalytics.org/) - self service platform for data scientists, developers and system operators.
* Roadmap: integration with [Apache Beam](http://beam.incubator.apache.org/), materializer for [Akka Streams](http://akka.io/).
* Oh, I forgot - chinese english is terrible.

### “An overview on optimization in Apache Hive: past, present, future” by Jesus Rodriguez (HortonWorks)

* Goals - sub-seconds latency, petabyte scale, ANSI SQL - never ending story.
* Metastore is often a bottleneck (DataNucleus ORM).
See also: 
[https://cwiki.apache.org/confluence/display/Hive/Design#Design-MetastoreArchitecture](https://cwiki.apache.org/confluence/display/Hive/Design#Design-MetastoreArchitecture)
* Metastore on HBase (Hive 2.x, alpha).
* [Cost Based Optimizer](https://cwiki.apache.org/confluence/display/Hive/Cost-based+optimization+in+Hive) (Apache Calcite), integrated from 0.14, enabled by default from 2.0.
* Optimizations: push down projections, push down filtering,
join reordering ([bushy joins](http://hortonworks.com/blog/hive-0-14-cost-based-optimizer-cbo-technical-overview/)), 
propagate projections, propagate filtering and more.
* New feature: materialized views. Cons: up-to-date statistics, optimizer could use view instead of table.
See more:
[HIVE-10459](https://issues.apache.org/jira/browse/HIVE-10459)
* Roadmap: optimizations based on CPU/MEM/IO costs.

### “Distributed in-database machine learning with Apache MADlib” by Roman Shaposhnik (Pivotal)

* Machine learning algorithms implemented as “distributed UDFs”.
* Works on PostgreSQL and its forks ([Pivotal Greenplum](http://greenplum.org/), [Apache HAWQ](http://hawq.incubator.apache.org/)).
* PivotalR + RPostgres for data scientists.
See more:
[https://cran.r-project.org/web/packages/PivotalR/PivotalR.pdf](https://cran.r-project.org/web/packages/PivotalR/PivotalR.pdf)
* Roadmap: more algorithms, execution on GPU with [CUDA](http://www.nvidia.com/object/cuda_home_new.html).

### “Interactive analytics at scale in Hive using Druid” by Jesus Rodriguez (HortonWorks)

* It works, at least on Jesus laptop with unreleased Hive version.
* Druid is used as a library (more or less).
* Druid indexing service is not used for indexing (regular Hive MR is used instead).
* Druid broker is used for querying but there are plans to bypass broker and access historical/realtime/indexing nodes directly. 
Right now the broker might be a bottleneck.
* [Apache Calcite](https://calcite.apache.org/docs/druid_adapter.html) is a key player in the integration.
* Pros: schema discovery.
* Cons: dims/metrics are inferred, right now there is no way to specify all important index details (e.g: time granularities).
* Roadmap: push down more things into the Druid for better query performance.
See more: 
[https://cwiki.apache.org/confluence/display/Hive/Druid+Integration](https://cwiki.apache.org/confluence/display/Hive/Druid+Integration)

### “Hadoop at Uber” by Mayank Basal (Uber)

* Fast pace of changes, respect!
* Shared cluster for batch jobs (YARN) and realtime jobs (Mesos) -> better resources utilization.
* [Apache Myriad](https://myriad.apache.org/) (YARN on Mesos): static allocation, no global quotas, no global priorities and many more limitations and problems.
* Unified Scheduler - just the name without any details yet.

### “Spark Performance” by Tim Ellison (IBM)

* Contribution to Spark in IBM way (closed solutions, heavily dependant on IBM JVM and IBM hardware).
* [Spark-kit](https://www.ibm.com/developerworks/java/jdk/spark/) (e.g custom block manager).
* Scala code (higher order functions) is extremely hard to optimize by JMV (e.g. using inlining)
* Mentioned benchmarks: [TCP-H](http://www.tpc.org/tpch/), [HiBench](https://github.com/intel-hadoop/HiBench)
* Remote Access Memory Access ([RDMA](https://en.wikipedia.org/wiki/Remote_direct_memory_access)) - zero copy networking
* Java Socket Over RDMA ([JSOR](http://www.ibm.com/developerworks/library/j-transparentaccel/))
* Coherent Accelerator Processor Interface (CAPI) to delegate processing to GPU
[https://en.wikipedia.org/wiki/Coherent_Accelerator_Processor_Interface](https://en.wikipedia.org/wiki/Coherent_Accelerator_Processor_Interface)
* Algebraic operations on GPU (performance gain only for huge matrices, there is a noticeable overhead when data is copied between CPU and GPU).
* And much more, low level technical details, look into the presentation by yourself:
[http://www.slideshare.net/JontheBeach/a-java-implementers-guide-to-boosting-apache-spark-performance-by-tim-ellison](http://www.slideshare.net/JontheBeach/a-java-implementers-guide-to-boosting-apache-spark-performance-by-tim-ellison)

### Apache Calcite and Apache Geode by Christian Tzolov (Pivotal)

* Apache Geode (AKA Gemfire) - distributed hashmap, consistent, transactional, partitioned, replicated, etc.
* [PDX serialization](https://cwiki.apache.org/confluence/display/GEODE/PDX+Serialization+Internals), on field level, type registry.
* Nested regions.
* Embeddable.
* Object Query Language (OQL) ~ SQL.
* Apache Calcite adapter (work in progress).
The adapter might be implemented gradually (from in-memory enumerable to advanced pushdowns/optimizations and bindable generated code).
* [Linq4j](https://github.com/apache/calcite/tree/master/linq4j) ported from .NET.

### “Data processing pipeline at Trivago” by Clemens Valiente (Trivago)

* Separated datacenters, REST collectors with HDD fallback, Apache Kafka, Camus, CSV.
* Hive MR jobs prepare aggregates and data subsets for Apache Impala, [Apache Oozie](http://oozie.apache.org/) used as scheduler.
* Problems with memory leaks in Apache Impala.
* [R/Shiny](http://shiny.rstudio.com/) connected to Impala for analytical purposes.
* Roadmap: [Kafka Streams](https://www.confluent.io/product/kafka-streams/), Impala + Kudu, Kylin + HBase.
* Interesting concept: direct access to Kafka Streams state (queryable [VoltDB](https://www.voltdb.com/)).

### “Implementing BigPetStore in Spark and Flink” by Marton Balasi (Cloudera)

* [BigTop](https://github.com/apache/bigtop) - way to build packages or setup big data servers and tools locally.
* [BigPetStore](https://github.com/apache/bigtop/tree/master/bigtop-bigpetstore) - generates synthethic data + sample stats calculation + sample recommendation (collaborative filtering).
* MR, Spark, [Flink](https://github.com/bigpetstore/bigpetstore-flink) implementations - nice method to learn Flink if you already know Spark.

### “Introduction to TensorFlow” by Gemma Parreno

* Global finalist of [NASA Space App Challenge](https://2016.spaceappschallenge.org/challenges/solar-system/near-earth-objects-machine-learning/projects/deep-asteriod), congrats!
* Extremely interesting session, but I’ve been totally lost - too much science :-(

### "Shared Memory Layer and Faster SQL for Spark Applications" by Dmitriy Setrakyan (GridGain)

* Apache Ignite is a in-memory data grid, compute grid, service grid, messaging and more.
* Off heap, [slab allocation](https://en.wikipedia.org/wiki/Slab_allocation)
* Run on [YARN](https://apacheignite.readme.io/docs/yarn-deployment)
* Run on [Mesos/Marathon](https://apacheignite.readme.io/docs/mesos-deployment)
* Shared, mutable, indexed, queryable RDD.
See more:
[https://ignite.apache.org/use-cases/spark/shared-memory-layer.html](https://ignite.apache.org/use-cases/spark/shared-memory-layer.html)
* In-memory MR (name node, job tracker and task trackers are totally bypassed).
See more: 
[https://ignite.apache.org/use-cases/hadoop/mapreduce](https://ignite.apache.org/use-cases/hadoop/mapreduce)
* HDFS cache (e.g: for caching hot data sets).
See more:
[https://ignite.apache.org/use-cases/hadoop/hdfs-cache](https://ignite.apache.org/use-cases/hadoop/hdfs-cache)
* Roadmap: dataframe compatible queries.

### "Apache CouchDB" by Jan Lehnardt (Neighbourhoodie Software)

* Master-master key-value store.
* HTTP as protocol.
* JSON as storage format.
* Simplified vector clock for conflicts handling (document hashes, consistent across cluster).
* Entertaining part about handling time in distributed systems, highly recommended.
* [Conflict-Free Replicated JSON Datatype](https://arxiv.org/pdf/1608.03960.pdf) was mentioned by Jan during talk, interesting.
* Roadmap: HTTP2, pluggable storage format, improved protocol for high latency networks.

### “Java memory leaks in modular environment” by Mark Thomas (Pivotal)

* Did you remember “OutOfMemoryError: PermGen space” in Tomcat? It is mostly not Tomcat fault.
* You should always set `-XX:MaxMetaspaceSize` in production systems.
* Excellent memory leaks analysis live demo using YourKit profiler (leaks in awt, java2d, rmi, xml).
[https://github.com/markt-asf/memory-leaks](https://github.com/markt-asf/memory-leaks)

### “Children and the art of coding” by Sebastien Blanc (RedHat)

* The best, entertaining session on the conference, IMHO! 
If you are happy parent, you should watch Sebastien's session (unfortunately sessions were not recorded, AFAIK).
* Logo, Scratch, Groovy, Arduino and [Makey Makey](http://makeymakey.com/)
