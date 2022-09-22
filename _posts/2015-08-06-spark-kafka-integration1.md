---
title: "Spark and Kafka integration patterns -- part 1"
date: 2015-08-06
tags: [Stream Processing, Apache Spark, Apache Kafka, Scala]
---

I published post on the [allegro.tech](http://allegro.tech/) blog, how to integrate Spark Streaming and Kafka.
In the blog post you will find how to avoid `java.io.NotSerializableException` exception
when Kafka producer is used for publishing results of the Spark Streaming processing.

[http://allegro.tech/spark-kafka-integration.html](http://allegro.tech/2015/08/spark-kafka-integration.html)

You could be also interested in the
[following part](http://mkuthan.github.io/blog/2016/01/29/spark-kafka-integration2/) of this blog post where
I presented complete library for sending Spark Streaming processing results to Kafka. 

Happy reading :-)
