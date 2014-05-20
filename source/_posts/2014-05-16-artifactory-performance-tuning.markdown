---
layout: post
title: "Artifactory Performance Tuning"
date: 2014-05-16
categories: [artifactory, jvm, performance, linux]
---

Few years ago I participated in Kirk Pepperdine Java performance tuning
training. One of the greatest technical training which I have ever been! And
also great opportunity to visit Crete :-) Let's check what I remember from the
training ...
  
In this blog post I would like to show _Artifactory_ memory utilization
analysis. Before you start any tuning you have to gather some statistics. You
cannot tune application if you don't know what should be improved.  
  
Get basic information about OS where _Artifactory_ is deployed and run:  
  
``` console
$cat /proc/cpuinfo | grep processor
processor       : 0
processor       : 1
processor       : 2
processor       : 3
```

``` console
$cat /proc/meminfo |grep MemTotal
MemTotal:      8163972 kB
```

Collect HTTP requests statistics from Apache logs (Apache is configured in front of _Artifactory_):  

{% img http://4.bp.blogspot.com/-6GBLIi2Guqo/U3Wyn9hcYvI/AAAAAAAAV68/SD_7RP3jcJE/s1600/screenshot2.png %}

Verify current JVM version and parameters:  

``` console
$java -version
java version "1.6.0_45"
Java(TM) SE Runtime Environment (build 1.6.0_45-b06)
Java HotSpot(TM) 64-Bit Server VM (build 20.45-b01, mixed mode)
```

``` console
-server -Xms2048m -Xmx2048m -Xss256k \
-XX:PermSize=256m -XX:MaxPermSize=256m \
-XX:NewSize=768m -XX:MaxNewSize=768m \
-XX:+UseParallelGC -XX:+UseParallelOldGC \
-XX:+PrintGCDetails -XX:+PrintGCTimeStamps -Xloggc:$CATALINA_HOME/logs/gc.log \
-Dartifactory.home=$ARTIFACTORY_HOME
```

* _-server_ Force server mode for VM (instead of client mode.
* _-Xms2048m -Xmx2048m_ Set up initial and total heap size to 2GB, it is recommended to set heap to fixed size. Default settings are too low for web applications (64MB if I remember).
* _-Xss256k_ Set stack memory size (256kB for each thread!)
* _-XX:PermSize=256m -XX:MaxPermSize=256m_ Set permanent generation size in similar way to heap size. Default settings are too low for web applications (64MB if I remember).
* _-XX:NewSize=768m -XX:MaxNewSize=768m_ One of the most important settings, increase young generation part of heap for short living objects, by default it is 1/4 or even 1/8 of total heap size. Web application creates a huge number of short living objects.
* _-XX:+UseParallelGC -XX:+UseParallelOldGC_ Utilize multiple CPUs for garbage collection, it should decrease GC time.
* _-XX:+PrintGCDetails -XX:+PrintGCTimeStamps -Xloggc:$CATALINA_HOME/logs/gc.log_ Log all GC activities for further analysis.

Check GC log statistics with HPjmeter:  

{% img http://2.bp.blogspot.com/-_GyvkTbLzhM/U3W4TETIadI/AAAAAAAAV7I/mSUz0SyHYOM/s1600/screenshot.png %}

In the summary tab, you can find the average interval between GCs and average
GC times. GC for young generation is called on every 95s for 35ms. Full GC
takes 2.25s, but fortunately it is called only ~three times per day (33 029
seconds). Great outcomes, you rather don't have a chance to recognize GC "stop
the world" pauses.  

{% img http://4.bp.blogspot.com/-Ydc1fO1ZefE/U3W4s-tciyI/AAAAAAAAV7Q/VUoj5cpXr_E/s1600/screenshot1.png %}

Based on the graph above, I could say that there is no memory leaks in
_Artifactory_. After each full GC memory is freed to the same level (more on
less). The peaks on the graph come from nightly jobs executed internally by
_Artifactory_ (indexer and backup).  

In general, heap size for young generation is enough to handle business hours
requests without full GC!!! And the GC pauses for young generation are very
short (30ms). No further _Artifactory_ tuning is needed :-)
