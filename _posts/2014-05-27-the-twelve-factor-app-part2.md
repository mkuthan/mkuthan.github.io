---
title: "The Twelve-Factor App - part 2"
date: 2014-05-27
tags: [Architecture]
---

This blog post is a continuation of [first part](http://mkuthan.github.io/blog/2014/05/26/the-twelve-factor-app-part1/) of this blog series.

## 7. Port Binding

> The twelve-factor app is completely self-contained and doesn't rely on runtime injection of a webserver into the execution environment to create a web-facing service.

I developed self-contained web application once, with embedded Jetty server. 
There are many product with embedded web server on the market, e.g: Artifactory.
Now most of my POC (Proof of Concept) use Spring Boot, when you can run web application as regular system process. 
After hell of JBoss class loader issues it seems to be the correct way to run the application.

## 8. Concurrency

> In the twelve-factor app, processes are a first class citizen.

Using processes instead of threads is controversial in JVM world.
But I agree that you can not scale out using threads only. 
What is also interesting, you should never daemonize process or write PID file. 
Just align to the system process management tools like upstart.

## 9. Disposability

> The twelve-factor app’s processes are disposable, meaning they can be started or stopped at a moment’s notice.

I faced disposability issues, when I was developing applications hosted on GAE (Google App Engine).
Forget about any heavy duty frameworks on GAE, the startup process must be really light. 
In general it's problematic in JVM world. 
Startup time of the JVM is significant itself, and JMV must spin up our application as well.
If I could compare JVM startup performance to the node.js there is a huge difference. 

I also remember how easily you can reconfigure system service when you can send -HUP signal. 
It would be nice to have this possibility for my applications.
 
## 10. Dev/prod parity

> The twelve-factor app is designed for continuous deployment by keeping the gap between development and production small.

Clear for me, test your production like environment as often as possible to minimize the risk. 
If the production database is Oracle, use Oracle XE for local development, not MySQL or H2.
If the production applications server is JBoss, use JBoss locally or Apache Tomcat at last resort.
Use the same JVM with similar memory settings if feasible.
If you deploy your application on Linux, do not use Windows for local development.
Virtualization or lightweight containers are your friends.
And so on ...

## 11. Logs

> A twelve-factor app never concerns itself with routing or storage of its output stream.

Hmm, I would prefer to use any logger (SLF4J) with configured appender instead of stdout. 
Instead file appender I could use Syslog appender and gather logs from all cluster nodes.
But maybe I'm wrong with this. I understand the point, than stdout is a basic common denominator for all runtime platform.

## 12. Admin processes

> Twelve-factor strongly favors languages which provide a REPL shell out of the box, and which make it easy to run one-off scripts.

For almost all my web application, I embedded BSH web servlet (Bean Shell Console). It rescued me out of trouble many times.
It isn't full fledged REPL like this one from Scala but still usable.
Oh, I forgot to mention about H2 web servlet, also embedded into most of my application.

Sometimes it's much easier to expose some admin functionality as JMX beans.
You can use Jolokia as REST JMX connector and easily prepare admin console using a few line of HTML and JavaScript. 

