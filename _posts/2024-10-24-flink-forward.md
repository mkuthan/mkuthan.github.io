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
I presented the notes as short checklists.
If an aspect was particularly appealing, I included a reference to supplementary materials.

![Intro](/assets/images/2024-10-24-flink-forward/intro.jpg)

## Keynotes

* Flink 2.0 [announced](https://www.ververica.com/blog/embracing-the-future-apache-flink-2.0) during the first day of the conference, nice timing.
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
As a seasoned [Dataflow Streaming Engine](https://medium.com/google-cloud/streaming-engine-execution-model-1eb2eef69a8e) user, I can say: _excellent move_.

![Disaggregated state storage](/assets/images/2024-10-24-flink-forward/disaggregated_state_storage.jpg)

* Streaming Lakehouse: An enabler for unified batch and streaming in an innovative way.
The engine decides whether to run in batch or streaming mode.

![Streaming lakehouse](/assets/images/2024-10-24-flink-forward/streaming_lakehouse.jpg)

* Breaking changes: While it's unfortunate that the _Scala API_ is deprecated, there's a new extension available: [Flink Scala API](https://github.com/flink-extended/flink-scala-api).

![Breaking changes](/assets/images/2024-10-24-flink-forward/breaking_changes.jpg)

## Flink autoscaling: A year in review - performance, challenges and innovations

* Autoscaling example, simplified but self-explanatory.

![Autoscaling example](/assets/images/2024-10-24-flink-forward/autoscaling_example.jpg)

* Challenges: Unfortunately, the speaker struggled with time management and couldn't delve into the details.
The key lesson for me: don't scale up if there is no effect of scaling.
Dataflow engineering team - can you hear me?

![Autoscaling challenges](/assets/images/2024-10-24-flink-forward/autoscaling_challenges.jpg)

* Memory management in Flink, for me looks like a configuration and tuning nightmare.
Flink autoscaling should help, see [FLIP-271](https://cwiki.apache.org/confluence/display/FLINK/FLIP-271%3A+Autoscaling).

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
* See also [6 Nines: How Stripe keeps Kafka highly available across the globe](https://www.confluent.io/events/kafka-summit-london-2022/6-nines-how-stripe-keeps-kafka-highly-available-across-the-globe/)

![Stripe](/assets/images/2024-10-24-flink-forward/stripe_kafka.jpg)

* Shared Zookeepers and shared Flink clusters can lead to issues with noisy neighbors and the propagation of failures. Extra operational costs are worth it to support system stability and performance.

## Visually diagnosing operator state problems

* TODO

## Zero interference and resource congestion in Flink clusters with Kafka data sources

* TODO

## Summary

Attending the conference was an interesting experience, providing valuable insights into the latest developments in Apache Flink.
Below, you can see the Gasometer Schöneberg on the EUREF-Campus, which served as the conference venue.

![Summary](/assets/images/2024-10-24-flink-forward/venue.jpg)
