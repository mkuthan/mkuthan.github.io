---
title: "Flink Forward 2024"
date: 2024-10-24
tags: [Apache Flink, Apache Kafka]
categories: [Conferences]
tagline: Berlin
header:
    overlay_image: /assets/images/2024-10-24-flink-forward/overlay.jpg
---

This week, I attended Flink Forward in Berlin, Germany.
The event celebrated the 10th anniversary of Apache Flink.
Below, you can find my overall impressions of the conference and notes from several interesting sessions.
If an aspect was particularly appealing, I included a reference to supplementary materials.

![Intro](/assets/images/2024-10-24-flink-forward/intro.jpg)

{: .notice--info}
I don't use Flink on a daily basis, but I hoped to gain some inspiration that I could apply to my real-time data pipelines running on GCP Dataflow.

## Keynotes

* Flink 2.0 [announced](https://www.ververica.com/blog/embracing-the-future-apache-flink-2.0) during the first day of the conference, perfect timing.
* Stephan Ewen with Feng Wang presented 15 years of the project history.
Apache Flink, which emerged around 2014, originally started as the Stratosphere project at German universities in 2009.

![Keynotes](/assets/images/2024-10-24-flink-forward/keynotes1.jpg)

* Kafka fails short in real-time streaming analytics, the statement I fully agree.

![Keynotes](/assets/images/2024-10-24-flink-forward/keynotes2.jpg)

* Truly unified batch and streaming with [Apache Paimon: Streaming Lakehouse](https://www.ververica.com/blog/apache-paimon-the-streaming-lakehouse).
* Fluss: Streaming storage for next-gen data analytics, it's going to be open-sourced soon.
Apologies for the low quality picture.

![Keynotes](/assets/images/2024-10-24-flink-forward/keynotes3.jpg)

## Revealing the secrets of Apache Flink 2.0

* Disaggregated state storage, goodbye [RocksDB](http://rocksdb.org/), introduce [Apache Celeborn](https://celeborn.apache.org).

![Disaggregated state storage](/assets/images/2024-10-24-flink-forward/disaggregated_state_storage.jpg)

* Streaming Lakehouse: An enabler for unified batch and streaming in an innovative way.
The engine decides whether to run in batch or streaming mode.

![Streaming lakehouse](/assets/images/2024-10-24-flink-forward/streaming_lakehouse.jpg)

* Breaking changes: While it's unfortunate that the _Scala API_ is deprecated, there's a new extension available: [Flink Scala API](https://github.com/flink-extended/flink-scala-api).
* PMC members mentioned that they're going to modernize the Java Streaming API soon. It's the oldest and hardest-to-maintain part of the Flink API.

![Breaking changes](/assets/images/2024-10-24-flink-forward/breaking_changes.jpg)

## Flink autoscaling: A year in review - performance, challenges and innovations

* Autoscaling example, simplified but self-explanatory.

![Autoscaling example](/assets/images/2024-10-24-flink-forward/autoscaling_example.jpg)

* Challenges: Unfortunately, the speaker struggled with time management and couldn't delve into the details.
The key lesson for me: don't scale up if there is no effect of scaling.
Dataflow engineering team - can you hear me?

![Autoscaling challenges](/assets/images/2024-10-24-flink-forward/autoscaling_challenges.jpg)

* Memory management in Flink, for me looks like a configuration and tuning nightmare.
Flink autoscaling should help, see: [FLIP-271](https://cwiki.apache.org/confluence/display/FLINK/FLIP-271%3A+Autoscaling).

![Memory model](/assets/images/2024-10-24-flink-forward/memory_model.jpg)

## Scaling Flink in the real world: Insights from running Flink for five years at Stripe

* The best session of the first day IMHO!
* I'm sure that Ben Augarten from Strip knows how to manage Flink clusters and jobs at scale.

![Stripe](/assets/images/2024-10-24-flink-forward/stripe_intro.jpg)

* With tight SLOs, there isn't time for manual operations.
If a job fails, roll back using the previously saved job graph.
How do you decide if a job fails in a generic way? You should listen to the session.

![Stripe](/assets/images/2024-10-24-flink-forward/stripe_rollbacks.jpg)

* Place a proxy in front of the Kafka cluster to ensure that jobs don't get stuck if a Kafka partition leader is unavailable.
See: [How Stripe keeps Kafka highly available across the globe](https://www.confluent.io/events/kafka-summit-london-2022/6-nines-how-stripe-keeps-kafka-highly-available-across-the-globe/)

![Stripe](/assets/images/2024-10-24-flink-forward/stripe_kafka.jpg)

* Shared Zookeepers and shared Flink clusters can lead to issues with noisy neighbors and the propagation of failures. Extra operational costs are worth it to support system stability and performance.

## Visually diagnosing operator state problems

* How to track the flow of data and identify where things go wrong?
* Can you inspect each late data record and figure out why it was late?
* Do you want to know what your state is before and after each step in your job?
* See also [The Murky Waters of Debugging in Apache Flink: Is it a Black Box?](https://datorios.com/blog/the-murky-waters-of-debugging-in-apache-flink/)
* Excellent logo, isn't it?

![Datorios](/assets/images/2024-10-24-flink-forward/flink_xray.jpg)

## Zero interference and resource congestion in Flink clusters with Kafka data sources

* One more session about current Kafka limitations.
* Noisy neighbours mitigation strategies in Kafka: quotas and cluster mirroring.
* Introduce WarpStream [Confluent has acquired WarpStream](https://www.confluent.io/blog/confluent-acquires-warpstream/).
* Stateless, leader-less brokers.
* Object storage for keeping state: expect higher latency, but it should be acceptable for most use cases.

![WarpStream](/assets/images/2024-10-24-flink-forward/warpstream.jpg)

## From Apache Flink to Restate - Event processing for analytics and Transactions

* A new business idea from one of the Flink founders: shift the focus from analytical to transactional processing.
* Apply resilience and consistency lessons learned from building Flink to distributed transactional, RPC-based applications.

![Restate Intro](/assets/images/2024-10-24-flink-forward/restate_intro.jpg)

* In simple terms, it resembles an orchestrated [Saga](https://blog.bytebytego.com/p/the-saga-pattern) pattern
* Durable and reliable async/await
* See [Why we built Restate](https://restate.dev/blog/why-we-built-restate/)

![Restate Durable Execution](/assets/images/2024-10-24-flink-forward/restate_durable_execution.jpg)

## Enabling Flink's Cloud-Native Future: Introducing ForSt DB in Flink 2.0

* The problem, local state doesn't fit cloud native architecture.

![Large state](/assets/images/2024-10-24-flink-forward/forst1.jpg)

* ForSt (for streaming) DB architecture

![ForSt DB architecture](/assets/images/2024-10-24-flink-forward/forst2.jpg)

* Performance dropped 100x when RocksDB replaced with object store as is.
* New async API, it requires changes in **all** Flink operators!

![State async API](/assets/images/2024-10-24-flink-forward/forst3.jpg)

* Asynchronous improves performance but introduces new challenge: ordering.

![Ordering](/assets/images/2024-10-24-flink-forward/forst4.jpg)

* Slower than local RocksDB but performance looks promising

![Benchmark](/assets/images/2024-10-24-flink-forward/forst5.jpg)

## Building Copilots with Flink SQL, LLMs and vector databases

* The most entertaining session of the conference, in my opinion.
* How to adopt real-time analysis for non-technical users?

![Real-time analysis adoption](/assets/images/2024-10-24-flink-forward/genai1.jpg)

* Steffen Hoellinger invited us to conduct a POC together with [Airy](https://airy.co/).

![Copilot architecture](/assets/images/2024-10-24-flink-forward/genai3.jpg)

* Hmm, some technical knowledge is still required 😀

![Sample session](/assets/images/2024-10-24-flink-forward/genai2.jpg)

* Key lesson: context is much more important than model.
* Keep small workspaces to avoid hallucinations.

![Context vs Model](/assets/images/2024-10-24-flink-forward/genai4.jpg)

* Flink SQL ML models, see: [FLIP-437](https://cwiki.apache.org/confluence/display/FLINK/FLIP-437%3A+Support+ML+Models+in+Flink+SQL).

## Materialized Table - Making Your Data Pipeline Easier

* The most eye opening session
* Batch, incremental and real-time unification
* Backfill

![Materialized Table](/assets/images/2024-10-24-flink-forward/materialized_table.jpg)

* Freshness vs Cost
* Apply `SET FRESHNESS = INTERVAL 1 HOUR` and framework will do the rest
* Support for most SQL queries (without `ORDER BY` cause)

![Freshness vs Cost](/assets/images/2024-10-24-flink-forward/freshness_vs_cost.jpg)

* Cool demo, materialized view freshness changed, Flink jobs rescheduled and BI dashboard updated in-place.
Yet another scenario for backfill.
* Community version coming soon, see [FLIP-435](https://cwiki.apache.org/confluence/display/FLINK/FLIP-435%3A+Introduce+a+New+Materialized+Table+for+Simplifying+Data+Pipelines).

## Event tracing

* Session based on IoT vehicle data in Mercedes-Benz.
* Apply [OpenTelemetry](https://opentelemetry.io/) for real-time data pipelines
* Tracing events sampling to avoid negative performance impact
* Kafka sources: extract tracing from headers
* Flink steps: attach spans to all events
* Kafka sinks: add tracing to headers
* In-summary: a lot of extra work

![Telemetry](/assets/images/2024-10-24-flink-forward/telemetry.jpg)

## Summary

Attending the conference was a valuable experience, offering deep insights into the latest developments in Apache Flink.
Here are my key takeaways:

* Listening to sessions about the challenges of Flink deployment and operations from the trenches made me appreciate the simplicity of [Dataflow](https://cloud.google.com/products/dataflow) even more.
* I now believe in the potential of truly unified batch and streaming processing.
FLIP-435 and the streaming lakehouse give hope that `SET FRESHNESS` could switch processing modes from batch, through incremental, to real-time.
* For high adoption of real-time analytics, consider using GenAI to hide the underlying data pipelines complexity.
* My general impression is that Kafka's limitations in the cloud-native era have been confirmed.

![Summary](/assets/images/2024-10-24-flink-forward/venue.jpg)
