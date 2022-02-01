---
title: "Spark application assembly for cluster deployments"
date: 2016-03-11
categories: [Apache Spark, Apache Kafka, Scala]
---

When I tried to deploy my first Spark application on a YARN cluster, 
I realized that there was no clear and concise instruction how to prepare the application for deployment.
This blog post could be treated as missing manual on how to build Spark application written in Scala to get deployable binary.

This blog post assumes that your Spark application is built with [SBT](http://www.scala-sbt.org/).
As long as SBT is a mainstream tool for building Scala applications the assumption seems legit.
Please ensure that your project is configured with at least SBT 0.13.6.
Open `project/build.properties` file, verify the version and update SBT if needed:

```
sbt.version=0.13.11
```


## SBT Assembly Plugin

The `spark-submit` script is a convenient way to launch Spark application on the YARN or Mesos cluster.
However, due to distributed nature of the cluster the application has to be prepared as single Java ARchive (JAR).
This archive includes all classes from your project with all of its dependencies.
This application assembly can be prepared using [SBT Assembly Plugin](https://github.com/sbt/sbt-assembly).

To enable SBT Assembly Plugin, add the plugin dependency to the `project/plugins.sbt` file:

```
addSbtPlugin("com.eed3si9n" % "sbt-assembly" % "0.14.1")
```

This basic setup can be verified by calling `sbt assembly` command. 
The final assembly location depend on the Scala version, application name and application version.
The build result could be assembled into `target/scala-2.11/myapp-assembly-1.0.jar` file.

You can configure many aspects of SBT Assembly Plugin like custom merge strategy
but I found that it is much easier to keep the defaults and follow the plugin conventions.
And what is even more important you don't have to change defaults to get correct, deployable application binary assembled by the plugin.

## Provided dependencies scope

As long as cluster provides Spark classes at runtime, Spark dependencies must be excluded from the assembled JAR.
If not, you should expect weird errors from Java classloader during application startup. 
Additional benefit of assembly without Spark dependencies is faster deployment.
Please remember that application assembly must be copied over the network to the location accessible by all cluster nodes (e.g: HDFS or S3).

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

The list of the Spark dependencies is always project specific.
SQL, Hive, MLib, GraphX and Streaming extensions are defined only for reference.
All defined dependencies are required by local build to compile code and run
[tests](http://mkuthan.github.io/blog/2015/03/01/spark-unit-testing/). 
So they could not be removed from the build definition in the ordinary way because it will break the build at all.

SBT Assembly Plugin comes with additional dependency scope "provided". 
The scope is very similar to [Maven provided scope](https://maven.apache.org/guides/introduction/introduction-to-dependency-mechanism.html).
The provided dependency will be part of compilation and test, but excluded from the application assembly.

To configure provided scope for Spark dependencies change the definition as follows:

```
val sparkVersion = "1.6.0"

"org.apache.spark" %% "spark-core" % sparkVersion % "provided",
"org.apache.spark" %% "spark-sql" % sparkVersion % "provided",
"org.apache.spark" %% "spark-hive" % sparkVersion % "provided",
"org.apache.spark" %% "spark-mlib" % sparkVersion % "provided",
"org.apache.spark" %% "spark-graphx" % sparkVersion % "provided",
"org.apache.spark" %% "spark-streaming" % sparkVersion % "provided",
"org.apache.spark" %% "spark-streaming-kafka" % sparkVersion
  exclude("log4j", "log4j")
  exclude("org.spark-project.spark", "unused"),
(...)
```

Careful readers should notice that "spark-streaming-kafka" dependency has not been listed and marked as "provided".
It was done by purpose because integration with Kafka is not part of Spark distribution assembly
and has to be assembled into application JAR.
The exclusion rules for "spark-streaming-kafka" dependency will be discussed later.

Ok, but how to recognize which libraries are part of Spark distribution assembly?
There is no simple answer to this question.
Look for `spark-assembly-*-1.6.0.jar` file on the cluster classpath,
list the assembly content and verify what is included and what is not.
In the assembly on my cluster I found core, sql, hive, mlib, graphx and streaming classes are embedded but not integration with Kafka.

```
$ tar -tzf spark-assembly-1.6.0.jar
META-INF/
META-INF/MANIFEST.MF
org/
org/apache/
org/apache/spark/
org/apache/spark/HeartbeatReceiver
(...)
org/apache/spark/ml/
org/apache/spark/ml/Pipeline$SharedReadWrite$$anonfun$2.class
org/apache/spark/ml/tuning/
(...)
org/apache/spark/sql/
org/apache/spark/sql/UDFRegistration$$anonfun$3.class
org/apache/spark/sql/SQLContext$$anonfun$range$2.class
(...)
reference.conf
META-INF/NOTICE
```

## SBT run and run-main

Provided dependency scope unfortunately breaks SBT `run` and `run-main` tasks.
Because provided dependencies are excluded from the runtime classpath, you should expect `ClassNotFoundException` during application startup on local machine.
To fix this issue, provided dependencies must be explicitly added to all SBT tasks used for local run, e.g.:

```
run in Compile <<= Defaults.runTask(fullClasspath in Compile, mainClass in(Compile, run), runner in(Compile, run))
runMain in Compile <<= Defaults.runMainTask(fullClasspath in Compile, runner in(Compile, run))
```

## How to exclude Log4j from application assembly?

Without Spark classes the application assembly is quite lightweight.
But the assembly size might be reduced event more!

Let assume that your application requires some logging provider.
As long as Spark internally uses Log4j, it means that Log4j is already on the cluster classpath.
But you may say that there is much better API for Scala than origin Log4j - and you are totally right.

The snippet below configure excellent Typesafe (Lightbend nowadays) [Scala Logging Library](https://github.com/typesafehub/scala-logging) dependency.

```
"com.typesafe.scala-logging" %% "scala-logging" % "3.1.0",

"org.slf4j" % "slf4j-api" % "1.7.10",
"org.slf4j" % "slf4j-log4j12" % "1.7.10" exclude("log4j", "log4j"),

"log4j" % "log4j" % "1.2.17" % "provided",
```

Scala Logging is a thin wrapper for SLF4J implemented using Scala macros.
The "slf4j-log4j12" is a binding library between SLF4J API and Log4j logger provider.
Three layers of indirection but who cares :-)
 
There is also top-level dependency to Log4J defined with provided scope. 
But this is not enough to get rid of Log4j classes from the application assembly.
Because Log4j is also a transitive dependency of "slf4j-log4j12" it must be explicitly excluded.
If not, SBT Assembly Plugin adds Log4j classes to the assembly even if top level "log4j" dependency is marked as "provided".
Not very intuitive but SBT Assembly Plugin works this way.

Alternatively you could disable transitive dependencies for "slf4j-log4j12" at all.
It could be especially useful for libraries with many transitive dependencies which are expected to be on the cluster classpath.

```
"org.slf4j" % "slf4j-log4j12" % "1.7.10" intransitive()
```

## Spark Streaming Kafka dependency

Now we are ready to define dependency to "spark-streaming-kafka".
Because Spark integration with Kafka typically is not a part of Spark assembly,
it must be embedded into application assembly.
The artifact should not be defined within "provided" scope.

```
val sparkVersion = "1.6.0"

(...)
"org.apache.spark" %% "spark-streaming-kafka" % sparkVersion
  exclude("log4j", "log4j")
  exclude("org.spark-project.spark", "unused"),
(...)
```

Again, "log4j" transitive dependency of Kafka needs to be explicitly excluded. 
I also found that marker class from weird Spark "unused" artifact breaks default SBT Assembly Plugin merge strategy.
It is much easier to exclude this dependency than customize merge strategy of the plugin.

## Where is Guava?

When you look at your project dependencies you could easily find Guava (version 14.0.1 for Spark 1.6.0).
Ok, Guava is an excellent library so you decide to use the library in your application.

*WRONG!*

Guava is on the classpath during compilation and tests but at runtime you will get "ClassNotFoundException" or method not found error.
First, Guava is shaded in Spark distribution assembly under `org/spark-project/guava` package and should not be used directly.
Second, there is a huge chance for outdated Guava library on the cluster classpath. 
In CDH 5.3 distribution, the installed Guava version is 11.0.2 released on Feb 22, 2012 - more than 4 years ago!
Since the Guava is [binary compatible](http://i.stack.imgur.com/8K6N8.jpg) only between 2 or 3 latest major releases it is a real blocker.

There are experimental configuration flags for Spark `spark.driver.userClassPathFirst` and `spark.executor.userClassPathFirst`.
In theory it gives user-added jars precedence over Spark's own jars when loading classes in the the driver.
But in practice it does not work, at least for me :-(.

In general you should avoid external dependencies at all cost when you develop application deployed on the YARN cluster.
Classloader hell is even bigger than in JEE containers like JBoss or WebLogic.
Look for the libraries with minimal transitive dependencies and narrowed features.
For example, if you need a cache, choose [Caffeine](https://github.com/ben-manes/caffeine) over Guava. 

## Deployment optimization for YARN cluster

When application is deployed on YARN cluster using `spark-submit` script, 
the script upload Spark distribution assembly to the cluster during every deployment.
The distribution assembly size is over 100MB, ten times more than typical application assembly!

So I really recommend to install Spark distribution assembly on well known location on the cluster 
and define `spark.yarn.jar` property for `spark-submit`.
The assembly will not be copied over the network during every deployment.

```
spark.yarn.jar=hdfs:///apps/spark/assembly/spark-assembly-1.6.0.jar
```

## Summary

I witnessed a few Spark projects where `build.sbt` were more complex than application itself.
And application assembly was bloated with unnecessary 3rd party classes and deployment process took ages.
Build configuration described in this blog post should help you deploy Spark application on the cluster smoothly
and still keep SBT configuration easy to maintain.
