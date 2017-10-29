---
layout: post
title: "Kafka Streams DSL vs Processor API"
date: 2017-11-04
comments: true
categories: [kafka, streaming, scala]
---

Kafka Streams is a Java library and can be used to build real-time, highly scalable, fault tolerant,
distributed applications.
Kafka Streams provides three different ways for applications development:

* Kafka Streams DSL (Domain Specific Language) recommended way for most users 
because business logic can be expressed in a few lines of code.
All stateless and stateful transformations are defined using declarative, 
functional programming style (filter, map, flatMap, reduce, aggregate operations).
Kafka Stream DSL encapsulates most of the stream processing complexity
but unfortunately it also hides many useful knobs and switches. 

* Kafka Processor API provides low level, imperative way to define stream processing logic.
At first sight Processor API could look hostile but gives much more flexibility to developer.
This blog post shows that hand crafted stream processing might be a magnitude more efficient than
naive implementation using Kafka DSL.

* KSQL is a promise that stream processing could be expressed by anyone using SQL like language.
It offers an easy way to express stream processing transformations as an alternative to writing 
an application in a programming language such as Java.
In addition processing transformation written in SQL like language can be highly optimized 
by execution engine without developer effort. 
Unfortunately KSQL was released recently and it is at very early development stage.

In the first part of this blog post I'll define simple but still realistic business use case to solve.
Then you will learn how to implement this use case with Kafka Stream DSL 
and how much the processing performance is affected by this naive solution.
At this moment you could stop reading and scale-up Kafka cluster ten times to fulfill business requirements 
or you could continue and learn how to optimize the processing with low level Kafka Processor API.

## Business Use Case

TODO

## Kafka Stream DSL

TODO

## Under the hood

TODO

## Kafka Processor API

TODO

## Summary

TODO