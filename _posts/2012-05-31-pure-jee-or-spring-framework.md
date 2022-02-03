---
title: "Pure JEE or Spring Framework"
date: 2012-05-31
categories: [Architecture, Java, Spring]
---

During my career as J2EE and JEE software developer I have been trying to use pure JEE two o three times. 
And I decided to do not repeat this exercise any more, it would be waste of my precious time. 
  
Below you can find short but quite comprehensive summary (based on [Ilias Tsagklis](http://www.javacodegeeks.com/2012/05/why-i-will-continue-to-use-spring-and.html)):

>  The strength of Java EE is the open standard established in a documented process by various parties. For this reason building applications for this platform is very popular, although many projects use only a little amount of Java EE API. Another important fact to notice is that every (wise) Java packaged software supplier supports the major Java EE platforms. As a result of all this many larger enterprises host Java EE servers in house anyway. Running Spring applications on Java EE servers is near-by then. This setup may provide benefits for enterprises with many dozens or even hundrets of Java applications in production.
>
> 1. Migrating JEE servers is considerably easier because the applications use less server API *directly*.
> 2. For the sake of little migration costs, most of the in-house clients will decide to migrate their business applications to current server versions.
> 3. Less server generations in IT landscape, less local development environment generations due to little Java versions in production, simpler ALM solutions -- all in all: manageable complexity and more homogeneous landscape.
> 4. Fast response to new client requirements: if you need new (Spring) features you'll only compile a new version of WAR/EAR files and deploy the stuff to an arbitrary Java runtime.
> 5. The range of potential target runtime environments will be higher compared to Java EE full stack. Which means you may be "more" plattform independent.
> 6. With Spring you can add smaller peaces of functionality to your applications or server environments (avoids: Snowball-effect).
> 7. Compared to Java EE, the complete innovation cycle is faster (from the feature request to the usage in production).
> 8. Spring enhancements are made according to actual real world client project requirements which ensures their practical applicability.
> 9. Application development teams remain responsible for the application development stack and can decide flexibly which API fits the clients needs.

> It's difficult to achieve all these benefits in a pure Java EE development stack (some of them address conceptual problems in Java EE). One possible option may be to modularize JEE application server architecture. Modularization as a standard, like in Java SE. It may also be valid to think about the release processes (i.e. JCP).

I would add my $.02:

* Not everything is portable across AS (or different version of AS from single vendor). 
I migrated JEE applications several times, mainly due to AS end of support. Spring framework provides necessary abstraction.

* JEE lifecycle is extremely long, you have to wait for the new standard, than for the vendor to apply 
the changes to the AS, than for the infrastructure team to engineer new AS version. Based on my corporate experience, 
it takes ~4 years from the initial release provided by Spring Framework.
 
* Developer productivity is higher if they can run application on the lightweight container like Jetty or Tomcat.
You can even consider to avoid Servlet container at all and run your application as regular system process.

* Spring Framework enhancements are driven by real use cases not by vendor marketing team or mad scientist. 
Did you try to use JPA2 Criteria? Or maybe do you prefer Query DSL instead?

* One size does not fit all, that's the point. Pure JEE is very limited but Spring Framework ecosystem is extraordinary rich.

I attended JEE evangelists sessions on several conferences around the world. And I can confirm, they did great speaks. 
It seams that pure JEE works for them, what is strange that it does not work for other specialists I know ;-)  