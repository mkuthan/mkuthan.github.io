---
title: "GCP FinOps for Data Pipelines"
date: 2022-04-02
categories: [GCP, FinOps]
tagline: "Streaming and batch use cases"
header:
    overlay_image: /assets/images/katie-harp-w45gZMWrJWc-unsplash.webp
    overlay_filter: 0.2
---

Do you monitor costs of the data pipelines in exactly the same way as you monitor the processing job health, latency or throughput?
Nowadays, taking care of costs efficiency is an integral part of every data engineer job.
I would like to share my own experiences with applying [FinOps](https://www.finops.org/introduction/what-is-finops/) discipline 
in organization within tens of data engineering teams and thousands of data pipelines.

## Overview

FinOps is a very broad term, but what is especially important from a data engineer perspective?

* You take ownership of the cloud usage and costs
* Cost is introduced as a regular metric of every data pipeline
* To optimize the costs you have to know detailed billing of every cloud resource  
* Cost analysis becomes a part of [definition of done](https://www.agilealliance.org/glossary/definition-of-done/)
* Premature costs optimization is the root of all evil (do you remember Donald Knuth paper about [performance](https://wiki.c2.com/?PrematureOptimization)?)

All my blog posts are written based on my own experiences, everything is battle tested and all presented screens are taken from the real systems.
And this time it will not be different, I will share how to keep costs of the streaming and batch pipelines on Google Cloud Platform under control.
On daily basis I manage the analytical data platform, the project with $100k+ monthly budget.
I'm also early Google Cloud Platform adopter and early FinOps practitioner in the organization, so
at the end of the post you will also learn how to scale the methodology across the teams and departments. 

## Google Cloud Platform resources

You may be surprised how complex is Google Cloud Platform billing.
There are many [products](https://cloud.google.com/products), every product has many [SKUs](https://cloud.google.com/skus), a dozen of different elements to track and count.
What are the most important cost factors in the Google Cloud Platform for data pipelines?

* Common resources
    - vCPU time
    - Memory
    - Persistent disks (standard or SSD)
    - Local disks (standard or SSD)
    - Network egress 
* Dataflow
    - Streaming service (realtime pipelines)
    - Shuffle service (batch pipelines)
* Dataproc
    - Licensing Fee
* Cloud Storage
    - Standard, Nearline, Coldline, Archive Storage
    - Class A operations
    - Class B operations
    - Data retrieval
    - Early deletion
* BigQuery
    - Storage (Active, Long Term, Physical)
    - Analysis
    - Streaming inserts
    - Storage API
* Pubsub 
    - Message delivery basic
    - Inter-region data delivery
    - Topic/Subscription message backlog
* Cloud Composer
    - SQL vCPU time
    - Environment fee
* Monitoring
    - Metric Volume
    - Log Volume

There are more products also relevant to the data pipelines like [Vertex AI](https://cloud.google.com/vertex-ai),
[Bigtable](https://cloud.google.com/bigtable) (expensive beast), 
[Firestore](https://cloud.google.com/firestore) or 
[Memory Store](https://cloud.google.com/memorystore).
I do not use them on daily basis so they are out of this blog post scope.

## FinOps sources

Billing Export
BigQuery audit logs / information schema?

## Labelling convention

## Streaming use case

* pubsub topic
* pubsub subscriptions (regular + internal)
* dataflow
* logging

## Batch use case

* bigquery analysis
* bigquery storage api
* compute
* storage
* logging

## Scaling the discipline for the whole organization

## Summary

TODO
