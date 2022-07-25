---
title: "Apache Beam Summit 2022"
date: 2022-07-07
categories: [Conferences, Apache Beam]
tagline: "Austin"
header:
    overlay_image: /assets/images/mj-tangonan-wKfTNWaDYgs-unsplash.webp
    overlay_filter: 0.2
---

Last week I virtually attended [Apache Beam Summit 2022](https://2022.beamsummit.org) held in Austin, Texas.
The event concentrates around [Apache Beam](https://beam.apache.org/) 
and the [runners](https://beam.apache.org/documentation/runners/capability-matrix/) like Dataflow, Flink, Spark or Samza.
Below you can find my overall impression of the conference and notes from several interesting sessions.
If some aspect was particularly appealing I put the reference to supplementary materials. 

## Overview

The conference was a hybrid event, all sessions (except for workshops) were live-streamed for an online audience for free.

During the sessions you could ask the questions on the streaming platform, 
and the questions from the online audience were answered by the speakers.
When the sessions had finished, they were available on the streaming platform.
I was able to replay afternoon sessions the next day in the morning, very convenient for people in non US timezones like me. 

For the two days the program was organized within three tracks, the third day was dedicated to the onsite workshops
but there was also one track for the online audience.

## Highly recommended sessions

### Google's investment on Beam, and internal use of Beam at Google

[The keynote session](https://2022.beamsummit.org/sessions/google/) was presented by
Kerry Donny-Clark (manager of the Apache Beam team at Google).

* Google centric keynotes, I missed some comparison between Apache Beam / Dataflow and other data processing industry leaders
* A few words about new runners: Hazelcast, Ray, Dask
* TypeScript SDK as an effect of internal Google hackathon
* Google engagement in Apache Beam community

![Beam team at Google](/assets/images/beam_summit_google_team.png)

### How the sausage gets made: Dataflow under the covers

[The session](https://2022.beamsummit.org/sessions/dataflow-under-the-covers/) was presented by 
Pablo Estrada (software engineer at Google, PMC member).

* **Excellent session**, highly recommended if you deploy non-trivial Apache Beam pipelines on Dataflow runner
* What does "exactly once" really mean in Apache Beam / Dataflow runner

![Exactly once](/assets/images/beam_summit_exactly_once.png)

* Runner optimizations: fusion, flatten sinking, combiner lifting
* Interesting papers, they help to understand Dataflow runner principles: [Photon](https://research.google/pubs/pub41318/), [MillWheel](https://research.google/pubs/pub41378/)
* Batch vs streaming: same controller, but data path is different. Different engineering teams, no plans to unify.
* Batch bundle: 100-1000 elements vs. streaming bundle: < 10 elements (when pipeline is up-to-date)
* Controlling batches: [GroupIntoBatches](https://beam.apache.org/releases/javadoc/current/org/apache/beam/sdk/transforms/GroupIntoBatches.html)
* In streaming EACH element has a key (implicit or explicit)

![Elements keys in streaming](/assets/images/beam_summit_streaming_keys1.png)

* Processing is serial for each key!

![Elements keys in streaming](/assets/images/beam_summit_streaming_keys2.png)

### Introduction to performance testing in Apache Beam

[The session](https://2022.beamsummit.org/sessions/introduction-to-the-benchmarks-in-apache-beam/) was presented by 
Alexey Romanenko (principal software engineer at Talend, PMC member).

There are 4 performance tests categories in Apache Beam 
and the tests results are available at [https://metrics.beam.apache.org](https://metrics.beam.apache.org)

1. IO integration tests
    * only for batch
    * only for Java SDK
    * not for all IOs
    * only read/write time metrics
2. Core Beam tests
    * synthetic sources
    * ParDo, ParDo with SideInput, GBK, CoGBK
    * batch and streaming
    * all SDKs
    * Dataflow, Flink, Spark
    * many metrics
3. Nexmark
    * batch and streaming
    * Java SDK
    * Dataflow, Flink, Spark
    * small scale, not industry standard (hard to compare)
4. TPC-DS
    * industry standard;
    * SQL based
    * batch only
    * CSV and Parquet as input
    * Dataflow, Spark, Flink
    * 25 of 103 queries currently passes

### How to benchmark your Beam pipelines for cost optimization and capacity planning

[The session](https://2022.beamsummit.org/sessions/benchmark-pipelines/) was presented by
Roy Arsan (solution architect at Google).

* How to verify performance and costs for different Dataflow setups using predefined scenarios?
* Use [PerfKit](https://github.com/GoogleCloudPlatform/PerfKitBenchmarker) benchmark!
* Runtime, CPU utilization and costs for different workers

![WordCount PerfKit results for WordCount](/assets/images/beam_summit_perfkit1.png)

* Results for the streaming job scenario

![WordCount PerfKit results for streaming job](/assets/images/beam_summit_perfkit2.png)

### Palo Alto Networks' massive-scale deployment of Beam

[The session](https://2022.beamsummit.org/sessions/beam-palo-alto/) was presented by
Talat Uyarer (Senior Principal Software Engineer at Palo Alto Networks).

* 10k streaming jobs!
* Custom solution for creating / deploying / managing Dataflow jobs
* Custom solution for managing Kafka clusters
* Example job definition

![Job definition](/assets/images/beam_summit_paoalto_job_definition.png)

* Custom sinks (unfortunately no details)
* Rolling updates or fallback to drain/start (but with the job cold start to minimize downtime)
* Many challenges with Avro schema evolution, see [case study](https://beam.apache.org/case-studies/paloalto/)

### New Avro Serialization And Deserialization In Beam SQL

[The session](https://www.google.com/url?q=https://2022.beamsummit.org/sessions/avro-serialization-deserialization/) was presented by
Talat Uyarer (Senior Principal Software Engineer at Palo Alto Networks).

* How can we improve latency while using BeamSQL and Avro payloads?

![Beam SQL](/assets/images/beam_summit_avro_sql.png)

* Critical elements: Table Provider + Data to Beam Row converters
* New [Avro converter](https://github.com/talatuyarer/beam-avro-row-serializer) inspired by [RTB House Fast Avro](https://techblog.rtbhouse.com/2017/04/18/fast-avro/)
* The future

![Beam SQL](/assets/images/beam_summit_avro_future.png)

### RunInference: Machine Learning Inferences in Beam

[The session](https://www.google.com/url?q=https://2022.beamsummit.org/sessions/runinference/) was presented by
Andy Ye (software engineer at Google).

* Reusable transform to run ML inferences, see [documentation](https://beam.apache.org/documentation/sdks/python-machine-learning/)
* Support for PyTorch, SciKit and TensorFlow
* More in the future

![RunInference future](/assets/images/beam_summit_run_inference.png)

### Unified Streaming And Batch Pipelines At LinkedIn Using Beam

[The session](https://2022.beamsummit.org/sessions/unified-stream-and-batch-pipelines-at-linkedin-using-beam/) was presented by
Shangjin Zhang (staff software engineer at LinkedIn) and Yuhong Cheng (software engineer at Linkedin).

![LinkedIn kappa architecture](/assets/images/beam_summit_linkedin_kappa.png)

* Streaming back-filling issues : hard to scale, flood on lookup tables, noisy neighbor to regular streaming pipelines

![LinkedIn unified architecture](/assets/images/beam_summit_linkedin_unified.png)

* Single codebase: Samza runner for streaming, Spark runner for batch
* Unified PTransform with expandStreaming and expandBatch methods
* Unified table join: key lookup for streaming and coGroupByKey for batch
* No windows in the pipeline (lucky them)

![LinkedIn back-filling results](/assets/images/beam_summit_linkedin_results.png)

* Job duration decreased from 450 minutes (streaming) to 25 minutes (batch)

### Log ingestion and data replication at Twitter

[The session](https://2022.beamsummit.org/sessions/log-ingestion-replication-twitter/) was presented by
Praveen Killamsetti and Zhenzhao Wang (staff engineers at Twitter) .

Batch ingestion architecture - Data Lifecycle Manager:

![Twitter batch ingestion architecture](/assets/images/beam_summit_twitter_batch.png)

* Many data sources: HDFS, GCS, S3, BigQuery, [Manhattan](https://blog.twitter.com/engineering/en_us/a/2014/manhattan-our-real-time-multi-tenant-distributed-database-for-twitter-scale)
* Versioned datasets with metadata layer
* Replicates data to all data sources (seems to be very expensive)
* GUI for configuration, DLM manages all the jobs
* Plans: migrate 600+ custom data pipelines to DLM platform

Streaming log ingestion - [Sparrow](https://blog.twitter.com/engineering/en_us/topics/infrastructure/2022/twitter-sparrow-tackles-data-storage-challenges-of-scale) 

![Twitter log ingestion](/assets/images/beam_summit_twitter_streaming.png)

* E2E latency - up to 13 minutes instead of hours
* One Beam Job and Pubsub Subscription per dataset per transformation (again seems to be very expensive)
* Decreased job resource usage by 80-86%  via removing shuffle in BigQuery IO connector
* Reduced worker usage by 20% via data (batches) compression on Pubsub
* Optimized schema conversion logic (Thrift -> Avro -> TableRow)

### Detecting Change-Points in Real-Time with Apache Beam

[The session](https://2022.beamsummit.org/sessions/detecting-change-points-in-real-time/) was presented by
Devon Peticolas (principal engineer at Oden Technologies).

* Nice session with realistic, non-trivial streaming IOT scenarios
* How Oden uses Beam: detecting categorical changes based on continues metrics

![Oden use-case](/assets/images/beam_summit_oden_usecase.png)

* Attempt 1: stateful DoFn - problem: out of order events (naive current and last elements' comparison)

![Oden stateful DoFn](/assets/images/beam_summit_oden_stateful_dofn.png)

* Attempt 2: watermark triggered window - problem: lag for non-homogeneous data sources

![Oden watermark trigger](/assets/images/beam_summit_oden_trigger1.png)

* Attempt 3: data triggered window - problem: sparse data when not all events are delivered

![Oden data trigger](/assets/images/beam_summit_oden_trigger2.png)

* Smoothing, see the original session it is hard to summarize concept in the single sentence

![Oden smoothing](/assets/images/beam_summit_oden_smoothing.png)


## Worth seeing sessions

### Tailoring pipelines at Spotify

[The session](https://2022.beamsummit.org/sessions/spotify/) was presented by
Rickard Zwahlen (data engineer at Spotify).

* Thousands of relatively similar data pipelines to manage
* Reusable Apache Beam jobs packed as Docker images and managed by [Backstage](https://backstage.spotify.com/)
* Typical use cases: data profiling, anomaly detection
* More complex use cases - custom pipeline development

### Houston, we've got a problem: 6 principles for pipelines design taken from the Apollo missions

[The session](https://www.google.com/url?q=https://2022.beamsummit.org/sessions/6-principles-for-pipeline-design/) was presented by 
Israel Herraiz and Paul Balm (strategic cloud engineers at Google).

* Nothing spectacular but worth seeing, below you can find the agenda

![6 principles](/assets/images/beam_summit_6_principles.png)

### Strategies for caching data in Dataflow using Beam SDK

[The session](https://2022.beamsummit.org/sessions/strategies-for-caching-data-in-dataflow-using-beam-sdk/) was presented by
Zeeshan (cloud engineer).

* Side input (for streaming engine stored in BigTable)
* Util apache_beam.utils.shared.Shared (python only)
* Stateful DoFn (per key and window), define elements TTL for the global window
* External cache

### Migration Spark to Apache Beam/Dataflow and hexagonal architecture + DDD

[The session](https://www.google.com/url?q=https://2022.beamsummit.org/sessions/migration-spark/) was presented by Mazlum Tosun.

* DDD/Hexagonal architecture pipeline code organization
* Static dependency injection with [Dagger](https://www.google.com/url?q=https://github.com/google/dagger)
* Test data defined as JSON files

I would say: **overkill** ... 
I'm going to write the blog post on how to achieve testable Apache Beam pipelines aligned to DDD architecture in a simpler way :)

### Error handling with Apache Beam and Asgarde library
[The session](https://2022.beamsummit.org/sessions/error-handling-asgarde/) was presented by Mazlum Tosun.

* Functional way to handle and combine Failures, Beam native method is too verbose (try/catch, exceptionsInto, exceptionsVia)
* See [https://github.com/project-asgard/asgard](https://github.com/project-asgard/asgard)

Asgard error handling for Java:
![Asgard error handling for Java](/assets/images/beam_summit_asgard_java.png)

Asgard error handling for Python:
![Asgard error handling for Python](/assets/images/beam_summit_asgard_python.png)

### Beam as a High-Performance Compute Grid

[The session](https://2022.beamsummit.org/sessions/hpc-grid/) was presented by
Peter Coyle (Head of Risk Technology Engineering Excellence at HSBC) and Raj Subramani.

* Risk management system for investment banking at HSBC
* Flink runner & Dataflow runner
* Trillion evaluations (whatever it means)
* Choose the most efficient worker type for cost efficiency
* Manage shuffle slots quotas for batch Dataflow
* Instead of reservation (not available for Dataflow) define disaster recovery scenario and fallback to more common worker type

![Dataflow disaster recovery planning](/assets/images/beam_summit_hsbc.png)

### Optimizing a Dataflow pipeline for cost efficiency: lessons learned at Orange

[The session](https://2022.beamsummit.org/sessions/optimizing-cost-efficiency/) was presented by 
Jérémie Gomez (cloud consultant at Google) and Thomas Sauvagnat (data engineer at Orange).

* How to store [Orange LiveBox](https://en.wikipedia.org/wiki/Orange_Livebox) data into BigQuery (33TB of billed bytes daily)?
* Initial architecture: data on GCS, on-finalize trigger, Dataflow streaming job notified from Pubsub, read files from GCS and store to BQ
* It is not cheap stuff ;)
* The optimization plan

![Optimization plan](/assets/images/beam_summit_orange.png)

* Storage Write API instead on Streaming Inserts
* BigQuery batch loads did not work as well
* N2 workers instead of N1
* Smaller workers (n2-standard-8 instead of n2-standard-16)
* Disabled autoscaling (the pipeline latency is not so important, Dataflow autoscaler policy can not be configured)
* **USE BATCH INSTEAD OF STREAMING** if feasible

### Relational Beam: Process columns, not rows!

[The session](https://2022.beamsummit.org/sessions/relational-beam/) was presented by 
Andrew Pilloud and Brian Hulette (software engineers at Google, Apache Beam committers)

* Beam is not relational, is row oriented, data is represented as bytes
* What is needed: data schema, metadata of computation

![Relational](/assets/images/beam_summit_relational.png)

* Batched DoFn (does not exist yet) https://s.apache.org/batched-dofns
* Projection pushdown (currently for BigQueryIO.TypedRead only; 2.38 batch, 2.41 streaming)
* Do not use `@ProcessContext` - it deserializes everything, use `@FieldAccess` instead
* Use relational transforms: beam.Select, beam.GroupBy

### Scaling up pandas with the Beam DataFrame API

[The session](https://2022.beamsummit.org/sessions/scaling-up-pandas-with-the-beam-dataframe-api/) was presented by
Brian Hulette (software engineer at Google, Apache Beam committer).

* Nice introduction to Pandas
* DataframeTransform and DeferredDataFrame

![Dataframe transform](/assets/images/beam_summit_dataframe_transform.png)

* Dataframe code → Expression tree → Beam pipeline  
* Compliance with Pandas is limited as for now
* 14% of Pandas operations are order-sensitive (hard to distribute)
* Pandas [windowing operations](https://pandas.pydata.org/docs/user_guide/window.html) transformed into Beam windowing 
would be a real differentiator in the future

### Improving Beam-Dataflow Pipelines For Text Data Processing

[The session](https://2022.beamsummit.org/sessions/improving-beam-dataflow-pipelines-for-text-data-processing/)
by Sayak Paul and Nilabhra Roy Chowdhury (ML engineers at Carted)

* Use variable sequence lengths for sentence encoder model
* Sort data before batching
* Separate tokenization and encoding steps to achieve full parallelism
* More details in the [blog post](https://www.carted.com/blog/improving-dataflow-pipelines-for-text-data-processing/)

![Pipeline optimization at Carted](/assets/images/beam_summit_carted.png)

## Summary

I would thank organizers and speakers for the Beam Summit conference.
The project needs such events to share the knowledge of how leading organizations use Apache Beam, 
how to apply advanced data processing techniques and how the runners execute the pipelines.

Below you could find a few takeaways from the sessions:

* Companies like Spotify, Palo Alto Networks or Twitter develop custom, fully managed, declarative layers on top of Apache Beam. 
To run thousands of data pipelines without coding and excessive operations. 
* Streaming pipelines are sexy but much more expensive (and complex) than batch pipelines.
* Use existing tools like [PerfKit](https://github.com/GoogleCloudPlatform/PerfKitBenchmarker) for performance/cost evaluation. 
Check Apache Beam [performance metrics](https://metrics.beam.apache.org) to compare different runners (e.g. if you want to migrate from JDK 1.8 to JDK 11).
* Understand the framework and the runner internals, unfortunately it is necessary for troubleshooting. 
Be aware that Dataflow batch and streaming engines are developed by different engineering teams.
* Cloud resources are not infinite, be prepared and define disaster recovery scenarios (e.g. fallback to more common machine types).
* Future is bright: Beam SQL, integration with ML frameworks, column oriented vectorized execution, new runners
