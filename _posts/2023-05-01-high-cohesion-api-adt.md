---
title: "High cohesion API design with algebraic data type"
date: 2023-04-03
tags: [Software Engineering, Scala, Apache Beam]
header:
    overlay_image: /assets/images/2023-05-01-high-cohesion-api-adt/mourizal-zativa-OSvN1fBcXYE-unsplash.webp
    caption: "[Unsplash](https://unsplash.com/@mourimoto)"
---

API design is one of the more demanding software engineering field.
The API should be able to evolve and clients should be able to discover all functionalities without reading a documentation.
For strongly typed languages like Scala, all faulty API usages should generate compile time errors.

In this blog post I would focus on [cohesion](https://en.wikipedia.org/wiki/Cohesion_(computer_science),
or rather lack of cohesion in [Apache Beam](https://beam.apache.org/documentation/io/built-in/google-bigquery/)BigQuery I/O connector.
You will find elegant, strongly typed, adapter for the Apache Beam API implemented as [algebraic data types](https://en.wikipedia.org/wiki/Algebraic_data_type).

