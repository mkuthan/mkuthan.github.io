---
layout: post
title: "Long running Spark Streaming jobs on YARN cluster"
date: 2016-09-30
comments: true
categories: [spark, yarn, hdfs]
---

A long-running Spark Streaming job, once submitted to the YARN cluster should run forever until it is intentionally stopped.
Any interruption introduces substantial processing delays and could lead to data loss or duplicates.
Neither YARN nor Apache Spark have been designed for running long-running services.
But they have been successfully adapted to growing needs of near real-time processing implemented as long-running jobs.
Successfully does not necessarily mean without technological challenges.

This blog post summarizes my experiences in running mission critical, long-running Spark Streaming jobs on a secured YARN cluster.
You will learn how to submit Spark Streaming application to a YARN cluster to avoid sleepless nights during on-call hours.

## Fault tolerance

In the YARN cluster mode Spark driver runs in the Application Master, the first container allocated by the application.
This process is responsible for driving the application and requesting resources (Spark executors) from YARN.
What is important, Application Master eliminates need for any another process that run during application lifecycle.
Even if an edge Hadoop cluster node where the Spark Streaming job was submitted fails, the application stays unaffected. 

To run Spark Streaming application in the cluster mode, ensure that the following parameters are given to spark-submit command:

    spark-submit --master yarn --deploy-mode cluster

Because Spark driver and Application Master share a single JVM, any error in Spark driver stops our long-running job.
Fortunately it is possible to configure maximum number of attempts that will be made to re-run the application.
It is reasonable to set higher value than default 2 (derived from YARN cluster property ```yarn.resourcemanager.am.max-attempts```).
For me 4 works quite well, higher value may cause unnecessary restarts even if the reason of the failure is permanent.

    spark-submit --master yarn --deploy-mode cluster \
        --conf spark.yarn.maxAppAttempts=4

If the application runs for days or weeks without restart or redeployment on highly utilized cluster, 4 attempts could be exhausted in few hours. 
To avoid this situation, the attempt counter should be reset on every hour of so.

    spark-submit --master yarn --deploy-mode cluster \
        --conf spark.yarn.maxAppAttempts=4 \
        --conf spark.yarn.am.attemptFailuresValidityInterval=1h
 
Another important setting is a maximum number of executor failures before the application fails. 
By default it is ```max(2 * num executors, 3)```, well suited for batch jobs but not for long-running jobs.
The property comes with corresponding validity interval which also should be set.

    spark-submit --master yarn --deploy-mode cluster \
        --conf spark.yarn.maxAppAttempts=4 \
        --conf spark.yarn.am.attemptFailuresValidityInterval=1h \
        --conf spark.yarn.max.executor.failures={8 * num_executors} \
        --conf spark.yarn.executor.failuresValidityInterval=1h
        
For long-running jobs you could also consider to boost maximum number of task failures before giving up the job.
By default tasks will be retried 4 times and then job fails.

    spark-submit --master yarn --deploy-mode cluster \
        --conf spark.yarn.maxAppAttempts=4 \
        --conf spark.yarn.am.attemptFailuresValidityInterval=1h \
        --conf spark.yarn.max.executor.failures={8 * num_executors} \
        --conf spark.yarn.executor.failuresValidityInterval=1h \
        --conf spark.task.maxFailures=8
 
## Performance

When Spark Streaming application is submitted to the cluster, YARN queue where the job runs must be defined.
I strongly recommend to use YARN Capacity Scheduler and separate queue for long-running jobs.
Without a separate YARN queue your long-running job will be preempted by a massive Hive query sooner or later.

    spark-submit --master yarn --deploy-mode cluster \
        --conf spark.yarn.maxAppAttempts=4 \
        --conf spark.yarn.am.attemptFailuresValidityInterval=1h \
        --conf spark.yarn.max.executor.failures={8 * num_executors} \
        --conf spark.yarn.executor.failuresValidityInterval=1h \
        --conf spark.task.maxFailures=8 \
        --queue realtime_queue


Another important performance factor for Spark Streaming job is processing time predictability. 
Processing time should stay below batch time to avoid delays. 
I've found that Spark speculative execution helps a lot, especially on busy cluster. 
Batch processing times are much more stable than when speculative execution is disabled.
Unfortunately speculative mode can be enabled only if Spark actions are idempotent.

    spark-submit --master yarn --deploy-mode cluster \
        --conf spark.yarn.maxAppAttempts=4 \
        --conf spark.yarn.am.attemptFailuresValidityInterval=1h \
        --conf spark.yarn.max.executor.failures={8 * num_executors} \
        --conf spark.yarn.executor.failuresValidityInterval=1h \
        --conf spark.task.maxFailures=8 \
        --queue realtime_queue \
        --conf spark.speculation=true


## Security

On a secured HDFS cluster, long-running Spark Streaming jobs fails due to Kerberos ticket expiration.
Without additional settings, Kerberos ticket is issued when Spark Streaming job is submitted to the cluster.
When ticket expires Spark Streaming job is not able to write or read data from HDFS anymore.
 
In theory (based on documentation) it should be enough to pass Kerberos principal and keytab as spark-submit command:
 
    spark-submit --master yarn --deploy-mode cluster \
         --conf spark.yarn.maxAppAttempts=4 \
         --conf spark.yarn.am.attemptFailuresValidityInterval=1h \
         --conf spark.yarn.max.executor.failures={8 * num_executors} \
         --conf spark.yarn.executor.failuresValidityInterval=1h \
         --conf spark.task.maxFailures=8 \
         --queue realtime_queue \
         --conf spark.speculation=true \
         --principal user/hostname@domain \
         --keytab /path/to/foo.keytab

In practice, due to several bugs ([HDFS-9276](https://issues.apache.org/jira/browse/HDFS-9276), [SPARK-11182](https://issues.apache.org/jira/browse/SPARK-11182))
HDFS cache must be disabled. If not, Spark will not be able to read updated token from file on HDFS.

    spark-submit --master yarn --deploy-mode cluster \
         --conf spark.yarn.maxAppAttempts=4 \
         --conf spark.yarn.am.attemptFailuresValidityInterval=1h \
         --conf spark.yarn.max.executor.failures={8 * num_executors} \
         --conf spark.yarn.executor.failuresValidityInterval=1h \
         --conf spark.task.maxFailures=8 \
         --queue realtime_queue \
         --conf spark.speculation=true \
         --principal user/hostname@domain \
         --keytab /path/to/foo.keytab \
         --conf spark.hadoop.fs.hdfs.impl.disable.cache=true


## Logging

The easiest way for Spark jobs to access application logs is to configure Log4j console appender, 
wait for application termination and use ```yarn logs -applicationId [applicationId]``` command.
Unfortunately it is not feasible to terminate long-running Spark Streaming jobs to access the logs.

I recommend to install and configure Elastic, Logstash and Kibana (ELK stack).
ELK installation and configuration is out of this blog post scope,
but remember to log the following context fields:

* YARN application id
* YARN container hostname
* Executor id (Spark driver is always 000001, Spark executors start from 000002)
* YARN attempt (to check how many times Spark driver has been restarted, attempts are decreased during application lifecycle accordingly to ```spark.yarn.am.attemptFailuresValidityInterval```)

Log4j configuration with Logstash specific appender and layout definition should be passed to spark-submit command:
 
    spark-submit --master yarn --deploy-mode cluster \
         --conf spark.yarn.maxAppAttempts=4 \
         --conf spark.yarn.am.attemptFailuresValidityInterval=1h \
         --conf spark.yarn.max.executor.failures={8 * num_executors} \
         --conf spark.yarn.executor.failuresValidityInterval=1h \
         --conf spark.task.maxFailures=8 \
         --queue realtime_queue \
         --conf spark.speculation=true \
         --principal user/hostname@domain \
         --keytab /path/to/foo.keytab \
         --conf spark.hadoop.fs.hdfs.impl.disable.cache=true \
         --conf spark.driver.extraJavaOptions=-Dlog4j.configuration=file:log4j.properties \
         --conf spark.executor.extraJavaOptions=-Dlog4j.configuration=file:log4j.properties \
         --files /path/to/log4j.properties

Finally Kibana dashboard for Spark Job might look like:

{% img /images/blog/spark_job_logging.png [spark job logging] %}

## Monitoring

Long running job runs 24/7 so it is important to have an insight into historical metrics. 
Again, external tools are needed. 
I recommend to install Graphite for collecting metrics and Grafana for building dashboards.

First, Spark needs to be configured to report metrics into Graphite, prepare the ```metrics.properties``` file:

    *.sink.graphite.class=org.apache.spark.metrics.sink.GraphiteSink
    *.sink.graphite.host=[hostname]
    *.sink.graphite.port=[port]
    *.sink.graphite.prefix=stats.analytics // this prefix will be used later on
    
    driver.source.jvm.class=org.apache.spark.metrics.source.JvmSource
    executor.source.jvm.class=org.apache.spark.metrics.source.JvmSource

And configure spark-submit command:

    spark-submit --master yarn --deploy-mode cluster \
         --conf spark.yarn.maxAppAttempts=4 \
         --conf spark.yarn.am.attemptFailuresValidityInterval=1h \
         --conf spark.yarn.max.executor.failures={8 * num_executors} \
         --conf spark.yarn.executor.failuresValidityInterval=1h \
         --conf spark.task.maxFailures=8 \
         --queue realtime_queue \
         --conf spark.speculation=true \
         --principal user/hostname@domain \
         --keytab /path/to/foo.keytab \
         --conf spark.hadoop.fs.hdfs.impl.disable.cache=true \
         --conf spark.driver.extraJavaOptions=-Dlog4j.configuration=file:log4j.properties \
         --conf spark.executor.extraJavaOptions=-Dlog4j.configuration=file:log4j.properties \
         --files /path/to/log4j.properties:/path/to/metrics.properties

Spark publishes tons of metrics from driver and executors.
If I have to choose the most important one, it would be the last received batch records.
When ```StreamingMetrics.streaming.lastReceivedBatch_records == 0``` it probably means that Spark Streaming job has been stopped or failed.

Other important metrics are listed below:

When total delay is greater than batch interval, latency of the processing pipeline increases.

    driver.StreamingMetrics.streaming.lastCompletedBatch_totalDelay

When number of active tasks is lower than ```number of executors * number of cores```, allocated resources are not fully utilized.

    executor.threadpool.activeTasks

How much RAM is used for RDD cache.

    driver.BlockManager.memory.memUsed_MB

When there is not enough RAM for RDD cache, how much data has been spilled to disk. 
You should increase executor memory or change ```spark.memory.fraction``` Spark property to avoid performance degradation. 

    driver.BlockManager.disk.diskSpaceUsed_MB

What is JVM memory utilization on Spark driver.

    driver.jvm.heap.used
    driver.jvm.non-heap.used
    driver.jvm.pools.G1-Old-Gen.used
    driver.jvm.pools.G1-Eden-Space.used
    driver.jvm.pools.G1-Survivor-Space.used

How much time is spent on GC on Spark driver.

    driver.jvm.G1-Old-Generation.time
    driver.jvm.G1-Young-Generation.time

What is JMV memory utilization on Spark executors.

    [0-9]*.jvm.heap.used
    [0-9]*.jvm.non-heap.used
    [0-9]*.jvm.pools.G1-Old-Gen.used
    [0-9]*.jvm.pools.G1-Survivor-Space.used
    [0-9]*.jvm.pools.G1-Eden-Space.used

How much time is spent on GC on Spark executors.

    [0-9]*.jvm.G1-Old-Generation.time
    [0-9]*.jvm.G1-Young-Generation.time


When you configure first Grafana dashboard for Spark job, perhaps the first question will emerge: 

> How to configure Graphite query when metrics for every Spark application run are reported under its own application id?

For driver metrics use wildcard ```.*(application_[0-9]+).*``` and ```aliasSub``` Graphite function to present 'application id' as graph legend:

    aliasSub(stats.analytics.$job_name.*.prod.$dc.*.driver.jvm.heap.used, ".*(application_[0-9]+).*", "heap: \1")
    
For executor metrics again use wildcard ```.*(application_[0-9]+).*```, ```groupByNode``` Graphite function to sum metrics from all Spark executors and ```aliasSub``` Graphite function to present 'application id' as graph legend:

    aliasSub(groupByNode(stats.analytics.$job_name.*.prod.$dc.*.[0-9]*.jvm.heap.used, 6, "sumSeries"), "(.*)", "heap: \1")

Finally Grafana dashboard for Spark Job might look like:

{% img /images/blog/spark_job_monitoring.png [spark job monitoring] %}

If Spark application is restarted frequently, metrics for old, already finished runs should be deleted from Graphite.
Because Graphite does not compact inactive metrics, old metrics slow down Graphite itself and Grafana queries.

## Graceful stop

The last puzzle element is how to stop Spark Streaming application deployed on YARN in a graceful way.
The standard method for stopping (or rather killing) YARN application is using a command ```yarn application -kill [applicationId]```.
And this command stops the Spark Streaming application but the application might be killed in the middle of the batch.
So if the job reads data from Kafka, save processing results on HDFS and finally commit Kafka offsets
you should expect duplicated data on HDFS when job was stopped just before committing offsets.

The first attempt to solve graceful shutdown issue was to call Spark streaming context stop method in shutdown hook.

``` scala
sys.addShutdownHook {
    streamingContext.stop(stopSparkContext = true, stopGracefully = true)
}
```

Disappointingly a shutdown hook is called too late to finish started batch and Spark application is killed almost immediately.
Moreover there is no guarantee that a shutdown hook will be called by JVM at all.

At the time of writing this blog post the only confirmed way to shutdown gracefully Spark Streaming application on YARN
is to notify somehow the application about planned shutdown, and then stop streaming context programmatically (but not from shutdown hook).
Command ```yarn application -kill``` should be used only as a last resort if notified application did not stop after defined timeout.

The application can be notified about planned shutdown using marker file on HDFS (the easiest way), 
or using simple Socket/HTTP endpoint exposed on the driver (sophisticated way).

Because I like KISS principle, below you can find shell script pseudo-code for starting / stopping Spark Streaming application using marker file:

    start() {
        hdfs dfs -touchz /path/to/marker/file
        spark-submit ...
    }
    
    stop() {
        hdfs dfs -rm /path/to/marker/file
        force_kill=true
        application_id=$(yarn application -list | grep -oe "application_[0-9]*_[0-9]*"`)
        for i in `seq 1 10`; do
            application_status=$(yarn application -status ${application_id} 2>&1 | grep "State : \(RUNNING\|ACCEPTED\)")
            if [ -n "$application_status" ]; then
                sleep 60s
            else
                force_kill=false
                break
            fi
        done
        $force_kill && yarn application -kill ${application_id}
    }

In the Spark Streaming application, background thread should monitor ```/path/to/marker/file``` file, 
and when the file disappears stop the context calling ```streamingContext.stop(stopSparkContext = true, stopGracefully = true)```.

## Summary

As you could see, configuration for mission critical Spark Streaming application deployed on YARN is quite complex. 
It has been long, tedious and iterative learning process of all presented techniques by a few very smart devs. 
But at the end, long-running Spark Streaming applications deployed on highly utilized YARN cluster are extraordinarily stable.
