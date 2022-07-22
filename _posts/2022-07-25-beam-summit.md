---
title: "Apache Beam Summit 2022"
date: 2022-07-07
categories: [Conferences]
tagline: "Austin"
header:
    overlay_image: /assets/images/mj-tangonan-wKfTNWaDYgs-unsplash.webp
    overlay_filter: 0.2
---

Last week I virtually attended [Apache Beam Summit 2022](https://2022.beamsummit.org) held in Austin, Texas.
The event concentrates around [Apache Beam](https://beam.apache.org/) 
and the [runners](https://beam.apache.org/documentation/runners/capability-matrix/) like Dataflow, Flink, Spark or Samza.
Below you can find my overall impression on the conference and notes from several interesting sessions.
The notes are presented as short checklists, if some aspect was particularly appealing I put the reference to supplementary materials. 


## Highly recommended sessions

### How the sausage gets made: Dataflow under the covers

[The session](https://2022.beamsummit.org/sessions/dataflow-under-the-covers/) by Pablo Estrada (software engineer at Google, PMC member)

* **Excellent session**, highly recommended if you deploy non-trivial Apache Beam pipelines on Dataflow
* What doas "exactly once" really mean in Apache Beam / Dataflow runner

![Exactly once](/assets/images/beam_summit_exactly_once.png)

* Runner optimizations: fusion, flatten sinking, combiner lifting
* Interesting papers: [Photon](https://research.google/pubs/pub41318/), [MillWheel](https://research.google/pubs/pub41378/)
* Batch vs streaming: same controller, data path is different <- different engineering teams! No plans to unify ;)
* Batch bundle: 100-1000 elements vs. streaming bundle: < 10 elements (when pipeline is up-to-date)
* Controlling batches: [GroupIntoBatches](https://beam.apache.org/releases/javadoc/current/org/apache/beam/sdk/transforms/GroupIntoBatches.html)
* In streaming EACH element has a key (implicit or explicit)

![Elements keys in streaming](/assets/images/beam_summit_streaming_keys1.png)

* Processing is serial for each key!

![Elements keys in streaming](/assets/images/beam_summit_streaming_keys2.png)

### Google's investment on Beam, and internal use of Beam at Google

[The keynote session](https://2022.beamsummit.org/sessions/google/) by Kerry Donny-Clark (manager of the Apache Beam team at Google).

* Google centric keynotes, I missed some comparison between Apache Beam / Dataflow and other industry leaders
* A few words about new runners: Hazelcast, Ray, Dask
* TypeScript SDK as an effect of internal Google hackaton
* Google engagement in Apache Beam community

![Beam team at Google](/assets/images/beam_summit_google_team.png)

### RunInference: Machine Learning Inferences in Beam

[The session](https://www.google.com/url?q=https://2022.beamsummit.org/sessions/runinference/) by Andy Ye (software engineer at Google)

* Reusable transform to run ML inferences, see: [documentation](https://beam.apache.org/documentation/sdks/python-machine-learning/)
* Support for PyTorch, SciKit and TensorFlow
* More in the future

![RunInference future](/assets/images/beam_summit_run_inference.png)

### Introduction to performance testing in Apache Beam

[The session](https://2022.beamsummit.org/sessions/introduction-to-the-benchmarks-in-apache-beam/) by Alexey Romanenko (principal software engineer at Talend, PMC member)

There are 4 performance tests categories in Apache Beam.

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
    * Many metrics
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
    * 25 of 103 queries passes

Continuous performance tests results are available at: [https://metrics.beam.apache.org]

### How to benchmark your Beam pipelines for cost optimization and capacity planning

[The session](https://2022.beamsummit.org/sessions/benchmark-pipelines/) by Roy Arsan (solution architect at Google)

* [PerfKit](https://github.com/GoogleCloudPlatform/PerfKitBenchmarker) benchmark
* Easy way to verify performance and costs for different Dataflow setups using predefined scenarios
* Run time, CPU utilization and costs for different workers

![WordCount PerfKit results for WordCount](/assets/images/beam_summit_perfkit1.png)

* The scenario for job with Pubsub subscription

![WordCount PerfKit results for streaming job](/assets/images/beam_summit_perfkit2.png)

### Palo Alto Networks' massive-scale deployment of Beam

[The session](https://2022.beamsummit.org/sessions/beam-palo-alto/)
Talat Uyarer (Senior Principal Software Engineer at Palo Alto Networks)

* 10k streaming jobs!
* Custom solution for creating / deploying / managing Dataflow jobs
* Custom solution for managing Kafka clusters
* Example job definition

![Job definition](/assets/images/beam_summit_paoalto_job_definition.png)

* Custom sinks (unfortunately no details)
* Rolling updates or drain / start (but with the job cold start to minimize downtime)
* Many challenges with Avro schema evolution, see: [case study](https://www.google.com/url?q=https://beam.apache.org/case-studies/paloalto/)

### New Avro Serialization And Deserialization In Beam SQL

[The session](https://www.google.com/url?q=https://2022.beamsummit.org/sessions/avro-serialization-deserialization/) by
Talat Uyarer (Senior Principal Software Engineer at Palo Alto Networks)

* How can we improve latency while using BeamSQL and Avro payloads?

![Beam SQL](/assets/images/beam_summit_avro_sql.png)

* Table Provider + Data to Beam Row
* New [Avro converter](https://github.com/talatuyarer/beam-avro-row-serializer) inspired by [RTB House Fast Avro](https://techblog.rtbhouse.com/2017/04/18/fast-avro/)
* The future

![Beam SQL](/assets/images/beam_summit_avro_future.png)

## Worth seeing sessions

### Tailoring pipelines at Spotify

[The session](https://2022.beamsummit.org/sessions/spotify/) by Rickard Zwahlen (data engineer at Spotify)

* Thousands of data pipelines
* Reusable Apache Beam jobs managed by [Backstage](https://backstage.spotify.com/)
* For typical use cases: data profiling, anomaly detection, etc.

### Houston, we've got a problem: 6 principles for pipelines design taken from the Apollo missions

[The session](https://www.google.com/url?q=https://2022.beamsummit.org/sessions/6-principles-for-pipeline-design/) 
by Israel Herraiz and Paul Balm (strategic cloud engineers at Google)

* Nothing spectacular but worth seeing

![6 principles](/assets/images/beam_summit_6_principles.png)

### Strategies for caching data in Dataflow using Beam SDK

[The session](https://2022.beamsummit.org/sessions/strategies-for-caching-data-in-dataflow-using-beam-sdk/) by Zeeshan (cloud engineer)

* side input (for streaming engine stored in BigTable)
* apache_beam.utils.shared.Shared (python only)
* stateful DoFn (per key and window), define elements TTL for the global window
* external cache

### Migration Spark to Apache Beam/Dataflow and hexagonal architecture + DDD

[The session](https://www.google.com/url?q=https://2022.beamsummit.org/sessions/migration-spark/) by Mazlum Tosun (tech lead)

* DDD/Hexagonal architecture pipeline code organization
* Static dependency injection with [Dagger](https://www.google.com/url?q=https://github.com/google/dagger)
* Test data defined as JSON files

I would say: **overkill** ... 
I'm going to write the blog post how to achieve testable Apache Beam pipelines aligned to DDD architecture in a simpler way :)

### Beam as a High-Performance Compute Grid

[The session](https://2022.beamsummit.org/sessions/hpc-grid/) 
by Peter Coyle (Head of Risk Technology Engineering Excellence at HSBC) and Raj Subramani

* Risk management system for investment banking at HSBC
* Flink runner & Dataflow runner
* Trillion evaluations (whatever it means)
* Choose the most efficient worker type
* Manage shuffle slots quotas for Dataflow
* Instead of reservation (not available for Dataflow) define disaster recovery scenario and fallback to more common worker type

![Dataflow disaster recovery planning](/assets/images/beam_summit_hsbc.png)

### Optimizing a Dataflow pipeline for cost efficiency: lessons learned at Orange

[The session](https://2022.beamsummit.org/sessions/optimizing-cost-efficiency/) 
by Jérémie Gomez (cloud consultant at Google) and Thomas Sauvagnat (data engineer at Orange)

* How to store [Orange LiveBox](https://en.wikipedia.org/wiki/Orange_Livebox) data into BigQuery (33TB of billed bytes daily)?
* Initial architecture: data on GCS, on-finalize trigger, Dataflow streaming job notified from Pubsub, read files from GCS and store to BQ
* It is not cheap stuff ;)
* The optimization plan

![Optimization plan](/assets/images/beam_summit_orange.png)

* Storage Write API instead on Streaming Inserts or BQ loads
* N2 workers instead of N1
* Smaller workers (n2-standard-8 instead of n2-standard-16)
* Disabled autoscaling (the pipeline latency is not so important, Dataflow autoscaler policy can not be configured)
* **USE BATCH INSTEAD OF STREAMING**

## Summary

* Companies like Spotify, Palo Alto Networks or Twitter develop custom, fully managed, declarative layer on top of Apache Beam. 
To run thousands data pipelines without coding and excessive operations. 
* Use existing tools like [PerfKit](https://github.com/GoogleCloudPlatform/PerfKitBenchmarker) for performance/costs evaluation. 
Check Apache Beam [performance metrics](https://metrics.beam.apache.org) (e.g. if you want to migrate from JDK 1.8 to JDK 11).
* Cloud resources are not infinite, be prepared and define disaster recovery scenarios (e.g. fallback to more common machine types).
* Streaming pipelines are much more expensive (and complex) than batch pipelines.
* Understand the framework and the runner internals, unfortunately it is necessary for troubleshooting.
