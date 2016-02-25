---
layout: post
title: "Spark application assembly for cluster deployments"
date: 2016-03-01
comments: true
categories: [scala, sbt, spark, kafka]
---

The `spark-submit` script is a convenient way to to launch Spark application on the YARN or Mesos cluster.
However, due to distributed nature of the cluster the application has to be prepared as single Java ARchive (JAR).

When I tried deploy my first Spark application on the YARN cluster, 
I realized that there is no clear and concise instruction how to prepare the application for deployment.
This blog post could be treated as missing manual how to build Spark application written in Scala to get deployable binary.
You should expect a few tips and tricks from the trenches as well.

## SBT Assembly Plugin

This blog post assumes that your Spark application is built with [SBT](http://www.scala-sbt.org/).
As long as SBT is a mainstream tool for building Scala applications the assumption seems legit.

Because the latest [SBT assembly plugin](https://github.com/sbt/sbt-assembly) is an auto plugin, 
please ensure that your project is configured to use at least 0.13.6 SBT version.
Open `project/build.properties` file and verify the version:

```
sbt.version=0.13.11
```

To enable SBT assembly plugin, add the plugin dependency to the `project/plugins.sbt` file:

```
addSbtPlugin("com.eed3si9n" % "sbt-assembly" % "0.14.1")
```

This basic setup can be verified by calling `sbt assembly` command. 
The build result is located at `target/scala-2.11/myapp-assembly-1.0.jar` destination directory.
The final location and artifact name depend on the Scala version, application name and application version.

You can configure many aspects of SBT assembly plugin 
but I found that it is much easier use the defaults and follow the conventions.
And what is even more important you don't have to change defaults to get correct deployable binary assembled by the plugin.

## Provided dependencies scope

As long as cluster provides Spark classes at runtime, Spark dependencies must be excluded from the assembled JAR.
If not, you should expect weird errors from Java classloader during application startup. 
Additional benefit of assembly without Spark dependencies is faster deployment.
Please remember that application binary must be copied to the location accessible by cluster nodes over the network  (e.g: HDFS or S3).

Look at dependency section in your build file, it should look similar to the code snippet below:

```
val sparkVersion = "1.6.0"

"org.apache.spark" %% "spark-core" % sparkVersion,
"org.apache.spark" %% "spark-sql" % sparkVersion,
"org.apache.spark" %% "spark-hive" % sparkVersion,
"org.apache.spark" %% "spark-mlib" % sparkVersion,
"org.apache.spark" %% "spark-graphx" % sparkVersion,
"org.apache.spark" %% "spark-streaming" % sparkVersion,
"org.apache.spark" %% "spark-streaming-kafka" % sparkVersion,
(...)
```

The list of the Spark dependencies is always project specific, MLib or GraphX extensions are defined here as an example.
All defined dependencies are required by local build to compile code and 
[test application](http://mkuthan.github.io/blog/2015/03/01/spark-unit-testing/). 
So they could not be removed from the build definition in the ordinary way because it will break the build at all.
SBT assembly plugin comes with additional dependency scope "provided". 
The scope is very similar to [Maven provided scope](https://maven.apache.org/guides/introduction/introduction-to-dependency-mechanism.html).
The dependency will be part of compilation and test, but excluded from the runtime.

To configure provided scope for Spark dependencies change the definition as follows:

```
val sparkVersion = "1.6.0"

"org.apache.spark" %% "spark-core" % sparkVersion % "provided",
"org.apache.spark" %% "spark-sql" % sparkVersion % "provided",
"org.apache.spark" %% "spark-hive" % sparkVersion % "provided",
"org.apache.spark" %% "spark-mlib" % sparkVersion % "provided",
"org.apache.spark" %% "spark-graphx" % sparkVersion % "provided",
"org.apache.spark" %% "spark-streaming" % sparkVersion % "provided",
"org.apache.spark" %% "spark-streaming-kafka" % sparkVersion,
(...)
```

Careful readers should notice that "spark-streaming-kafka" dependency has not been marked as "provided".
It was done by purpose because integration with Kafka is not part of Spark distribution assembly
and has to assembled into application JAR.

Ok, but how to recognize what library is part of Spark distribution assembly and what is not?
Look for `spark-assembly-*-1.6.0.jar` on the cluster classpath,
and list the assembly and verify what is included.
In the assembly on my cluster I found core, sql, hive, mlib, graphx and streaming classes but not integration with Kafka.

```
$ tar -tzf spark-assembly-1.6.0.jar
META-INF/
META-INF/MANIFEST.MF
org/
org/apache/
org/apache/spark/
org/apache/spark/HeartbeatReceiver
(...)
org/apache/spark/deploy/yarn/ExecutorDelegationTokenUpdater$$anonfun$updateCredentialsIfRequired$1$$anonfun$apply$1.class
META-INF/maven/org.apache.spark/spark-yarn_2.10/
META-INF/maven/org.apache.spark/spark-yarn_2.10/pom.xml
META-INF/maven/org.apache.spark/spark-yarn_2.10/pom.properties
META-INF/services/org.apache.hadoop.fs.FileSystem
META-INF/services/com.fasterxml.jackson.core.JsonFactory
META-INF/services/com.fasterxml.jackson.databind.Module
META-INF/services/com.fasterxml.jackson.core.ObjectCodec
META-INF/services/tachyon.underfs.UnderFileSystemFactory
META-INF/services/org.apache.spark.sql.sources.DataSourceRegister
META-INF/services/javax.xml.bind.JAXBContext
META-INF/services/org.apache.commons.logging.LogFactory
reference.conf
META-INF/NOTICE
```

## SBT run and run-main

Provided dependency scope breaks SBT `run` and `run-main` tasks.
Because provided dependencies are excluded from the runtime, you should expect `ClassNotFoundException` during application startup on local machine.
To fix the issue, provided dependencies must be explicitly added to all tasks used for local run, e.g:

```
run in Compile <<= Defaults.runTask(fullClasspath in Compile, mainClass in(Compile, run), runner in(Compile, run)))
runMain in Compile <<= Defaults.runMainTask(fullClasspath in Compile, runner in(Compile, run)))
```

## How to exclude log4j from application assembly?

Without Spark classes the application assembly is quite lightweight.
But the assembly size might be reduced event more!

Let assume that your application requires some logging provider.
As long as Spark internally uses Log4j, it means that Log4j is already on the cluster classpath.
But you may say that there is much better API for Scala than origin Log4j - and you are totally right.

The snippet below configure excellent Typesafe (or Lightbend nowadays) [Scala Logging library](https://github.com/typesafehub/scala-logging) dependency.

```
"com.typesafe.scala-logging" %% "scala-logging" % "3.1.0",

"org.slf4j" % "slf4j-api" % "1.7.10",
"org.slf4j" % "slf4j-log4j12" % "1.7.10" exclude("log4j", "log4j"),

"log4j" % "log4j" % "1.2.17" % "provided",
```

Scala Logging is a thin wrapper for SLF4J implemented using Scala macros.
The "slf4j-log4j12" is a binding library between SLF4J API and Log4j logger provider.
 
There is also top-level dependency to Log4J defined with provided scope. 
But this is not enough to get rid of Log4j classes from the application assembly.
Because Log4j is also a transitive dependency of "slf4j-log4j12" it must be explicitly excluded.
If not, SBT assembly plugin adds Log4j classes to the assembly even if top level "log4j" dependency is marked as "provided".
Not very intuitive but SBT assembly plugin works this way.

Alternatively you could disable transitive dependencies for "slf4j-log4j12" at all.
It could be especially useful for libraries with many transitive dependencies which are expected to be on the cluster classpath.

```
"org.slf4j" % "slf4j-log4j12" % "1.7.10" intransitive()
```

## Where is Guava?

When you look at your project dependencies you will easily find Guava (version 14.0.1 for spark 1.6.0).
Ok, Guava is an excellent library so you decide to use the library in your application.

WRONG!

Guava is on the classpath during compilation and tests but at runtime you will get "ClassNotFoundException" or method not found error.
First, Guava is shaded in Spark distribution assembly under `org/spark-project/guava` package and should not be used directly.
Second, there is a huge chance for outdated Guava library on the cluster classpath. 
In CDH 5.3 distribution, the installed Guava version is 11.0.2 released on Feb 22, 2012 - more than 4 years ago!
Since the Guava is [binary compatible](http://i.stack.imgur.com/8K6N8.jpg) only with 2 or 3 latest major releases it is a real blocker. 

There are experimental configuration flags for Spark `spark.driver.userClassPathFirst` and `spark.executor.userClassPathFirst`.
In theory it gives user-added jars precedence over Spark's own jars when loading classes in the the driver.
In practice it does not work, at least for me :-(.

In general you should avoid external dependencies at all cost when you develop application deployed on the YARN cluster.
Classloader hell is even bigger than in JEE containers like JBoss or WebLogic.
Look for the libraries with minimal transitive dependencies and narrowed features.
For example, if you need a cache, choose [Caffeine](https://github.com/ben-manes/caffeine) over Guava. 

## Deployment optimization for YARN cluster

When application is deployed on YARN cluster using `spark-submit` script, 
the script upload spark distribution assembly to the cluster on every deployment.
The distribution assembly size is over 100MB, ten times more than typical application assembly!

So I really recommend to install Spark distribution assembly on well known location on the cluster 
and define `spark.yarn.jar` property for `spark-submit`.
The assembly will not be copied over the network during every deployment.

```
spark.yarn.jar=hdfs:///apps/spark/assembly/spark-assembly-1.6.0.jar
```

## Summary

I witnessed a few Spark projects where `build.sbt` was more complex than application itself.
And application assembly was bloated with unnecessary 3rd party classes and deployment process took ages.
Build configuration described in this blog post should help you deploy Spark application on the cluster smoothly
and still keep SBT definition configuration easy to maintain. 
