---
title: "GCP Cloud Composer 1.x Tuning"
date: 2022-03-16
categories: [GCP, Cloud Composer, Apache Airflow, Performance]
tagline: ""
header:
    overlay_image: /assets/images/aron-visuals-BXOXnQ26B7o-unsplash.webp
    overlay_filter: 0.2
---

I would love to develop only [streaming pipelines](/categories/stream-processing/) but in reality some of them are still batch oriented.
Today you will learn how to configure Google Cloud Platform scheduler - [Cloud Composer](https://cloud.google.com/composer) - 
in the performance and costs effective way.

The article is for *Cloud Composer* version `1.x` only.
Why not use version `2.x`? It is a very good question, indeed.
But the reality of real life has forced me to tune to the obsolete version.
{: .notice}

## Overview

*Cloud Composer* is a fully managed workflow orchestration service built on [Apache Airflow](https://airflow.apache.org).
I fully agree that *Cloud Composer* is a magnitude easier to set up than vanilla *Apache Airflow* but there are still some gotchas:

* How many resources are allocated by the single *Apache Airflow* task?
* How many concurrent tasks can be executed on the worker, what is a real worker capacity?
* What is a total parallelism of the whole *Cloud Composer* cluster?
* How to choose the right virtual machine type and how to configure *Apache Airflow* to fully utilize the allocated resources?
* What are the most important *Cloud Composer* metrics to monitor?

## Tasks

Let's begin with the *Apache Airflow* basic unit of work - [task](https://airflow.apache.org/docs/apache-airflow/stable/concepts/tasks.html).
There are two main kind of tasks: [operators](https://airflow.apache.org/docs/apache-airflow/stable/concepts/operators.html) and [sensors](https://airflow.apache.org/docs/apache-airflow/stable/concepts/sensors.html).
On my *Cloud Composer* installation operators are responsible for creating the ephemeral [Dataproc](https://cloud.google.com/dataproc) cluster and submit [Apache Spark](https://spark.apache.org) batch job to this cluster.
The sensors wait for [BigQuery](https://cloud.google.com/bigquery) data, the payload for the Spark jobs.
From the performance perspective, the operators are much more resource heavy than sensors.

The *BigQuery* sensors are short living tasks, the sensor checks for the data and if data exists the sensor quickly finishes.
If data is not available yet, the sensor also finishes, but it is also rescheduled for the next execution 
(see [poke_interval](https://airflow.apache.org/docs/apache-airflow/stable/_api/airflow/sensors/base/index.html#airflow.sensors.base.BaseSensorOperator) parameter).

The resources allocated by the *Spark* operator is kept for the whole *Spark* job's execution time.
It could take several minutes or even hours. 
For most of the time, the operator does not do much more than checking for the Spark job status, so it is a memory bound process.
Even if there is a CPU time slots shortage on the *Cloud Composer* worker, the negative impact on the Spark job itself is negligible.

So for further capacity planning we may count operators memory allocation and add some safety margin for the sensors.

How to check how much memory is allocated by the operators?
It's not easy task, the *Cloud Composer* workers form a [Kubernetes](https://kubernetes.io/) cluster.
You could try to connect to the Kubernetes *airflow-worker* [pod](https://kubernetes.io/docs/concepts/workloads/pods/) or ...
run a dozen of tasks and measure the real resource utilization.
I would opt for the second option.

![Cloud Composer worker memory usage](/assets/images/cloud_composer_worker_memory_usage.webp)

The workers' memory utilization increased from steady state 1.76GiB to 4.36GiB for 12 concurrent running operators.
The first insight in our tuning journey: every operator allocates approximately (4.36GiB - 1.76GiB) / 12 =~ 220MiB of RAM.

You should ask: Whaaaat 220MiB for the REST call to the *Dataproc* cluster API?
The architecture of the *Apache Airflow* is quite complex, every task is executed as [Celery worker](https://docs.celeryproject.org/en/stable/userguide/workers.html) with its own overhead.
Every task needs a connection to the *Apache Airflow* database, it also consumes resources.
Perhaps there are other factors I'm not even aware of ...

You should always measure the task memory usage by yourself, all further calculation heavily depends on it.
The memory usage might be varying for different *Apache Airflow* operators.
{: .notice}

## Worker Size

We have already known the memory utilization by the single task. Although, how many tasks can be executed concurrently on the worker?
It depends on:
* The type of *Apache Airflow* task (for my scenario - 220MiB)
* Allocatable memory on *Kubernetes* cluster
* The *Cloud Composer* built-in processes overhead
* The type of the virtual machine, finally

### Allocatable memory on *Kubernetes*

The allocatable memory on Kubernetes cluster is calculated in the [following way](https://cloud.google.com/kubernetes-engine/docs/concepts/cluster-architecture#memory_cpu):

* `Capacity - 25%` for the first 4GiB
* `Capacity - 20%` for the next 4GiB (up to 8GiB)
* `Capacity - 10%` for the next 8GiB (up to 16GiB)

So, for the standard virtual machines allocatable memory should be as follows:

| Worker         | Allocatable Memory                   |
| ---------------| -----------------------------------: |
| n1-standard-1  | 3.75GiB - 25% = 2.8GiB               |
| n2-standard-2  | (4GiB - 25%) + (4GiB - 20%) = 6.2GiB |
| n2-highmem-2   | 6.2GiB + (8GiB - 10%) = 13.4GiB      |

How it looks in practice? It is always worth to check because the real allocatable memory is lower than in the calculations :)

*n1-standard-1* virtual machines: 2.75GB (**2.56GiB**)
![Cloud Composer nodes for n1-standard-1](/assets/images/cloud_composer_nodes_n1_standard_1.webp)

*n2-standard-2* virtual machines: 6.34GB (**5.9GiB**)
![Cloud Composer nodes for n2-standard-2](/assets/images/cloud_composer_nodes_n2_standard_2.webp)

Thank you, Google, for using different units across the console. 
An intellectual challenge every time when I have to convert GB to GiB and vice-versa.
{: .notice}

Please keep in mind that minimal *Cloud Composer* cluster has to have three workers.
And the workers is only a part of total *Cloud Composer* [costs](https://cloud.google.com/products/calculator#tab=composer).
Below you can find the estimated monthly costs for 3-nodes *Cloud Composer* installation in eu-west1 region.
You can assume that real usage when *Cloud Composer* schedules real tasks are ~20% higher than presented numbers.

| Worker        | Estimated monthly cost (eu-west1) |
| ------------- | --------------------------------: |
| n1-standard-1 | ~ $410                            |
| n2-standard-2 | ~ $510                            |
| n2-highmem-2  | ~ $570                            |

### *Cloud Composer* overhead

There are also many built-in *Cloud Composer* processes run on every worker.
As long as the *Cloud Composer* is a managed service, you don't have control over these processes.
Or even if you know how to hack some of them, you should not - the future upgrades or the troubleshooting will be a bumpy walk.

![Cloud Composer pods](/assets/images/cloud_composer_pods.webp)

Do not rely on the reported requested memory, it is just a garbage. 
Just measure the maximum worker memory utilization on the clean *Cloud Composer* installation and add the result to the final estimate.

![Cloud Composer memory overhead](/assets/images/cloud_composer_memory_overhead.webp)

My measures show 1.6GiB of RAM for built-in *Cloud Composer* processes on every worker.

### Tasks' space

Now we are ready to estimate available memory and the maximum number of concurrent tasks.

| Worker        |                  Available Memory |          Maximum Tasks |
| ------------- |----------------------------------:|-----------------------:|
| n1-standard-1 |  3 * (2.56GiB - 1.6GiB) = 2.88GiB |  2.88GiB / 220MiB = 13 |
| n2-standard-2 |   3 * (5.9GiB - 1.6GiB) = 12.9GiB |  12.9GiB / 220MiB = 60 |
| n2-highmem-2  | 3 * (~13.4GiB - 1.6GiB) = 35.4GiB | 35.4GiB / 220MiB = 164 |

I would also recommend making some reservation if you want the stable environment without unexpected incidents during your on-duty shift.
For 20% reservation, the *Cloud Composer* cluster capacity will be defined as follows:

| Worker         | Maximum Tasks |
| ------------- :|--------------:|
| n1-standard-1  |            10 |
| n2-standard-2  |            48 |
| n2-highmem-2   |           131 |


## *Apache Airflow* tuning

### Parallelism and worker concurrency

When the maximum number of tasks is known, it must be applied manually in the *Apache Airflow* configuration.
If not, *Cloud Composer* sets the defaults and the workers will be under-utilized or *airflow-worker* pods will be [evicted](https://cloud.google.com/composer/docs/how-to/using/troubleshooting-dags) due to memory overuse.

* `core.parallelism` - The maximum number of task instances that can run concurrently in Airflow regardless of worker count, 18 if not specified explicitly.
* `celery.worker_concurrency` - Defines the number of task instances that a worker will take, 6 for 3-nodes cluster if not specified explicitly.

In my 3-workers cluster scenario the following settings should be applied.

| Worker        | core.parallelism | celery.worker_concurrency |
| ------------- |-----------------:|--------------------------:|
| n1-standard-1 |               10 |                       3~4 |
| n2-standard-2 |               48 |                        16 |
| n2-highmem-2  |              131 |                        43 |

As you can see, *Cloud Composer* defaults do not match any of the presented worker types.

### Scheduler

I have also found two another *Apache Airflow* properties worth to modify: 

* `scheduler.min_file_process_interval` - The minimum interval after which a DAG file is parsed and tasks are scheduled, 0 if not specified.
* `scheduler.parsing_processes` - Defines how many DAGs parsing processes will run in parallel, 2 if not specified.

I highly recommend to set `scheduler.min_file_process_interval` to at least 30 seconds to avoid high CPU usage on worker where *airflow-scheduler* pod is running.
The [scheduler](https://airflow.apache.org/docs/apache-airflow/stable/concepts/scheduler.html) is running only on the single worker,
so it heavily impacts the other processes and tasks on this worker.

The `scheduler.parsing_processes` should be set to `max(1, number of CPUs - 1)`, set to 1 unless you define workers with 4 CPU or more.
Again it should lower the CPU utilization on the worker when *airflow-scheduler* is running.


## Monitoring

pod evictions / failures
dag parsing time
memory utilization (does it close to the limits)
cpu utilization (is it evenly distributed across the workers)

## Summary

TODO

Last but not least, I would like to thank Piotrek, Mariusz, Paweł, Patryk and Michał
for the fruitful discussions.
