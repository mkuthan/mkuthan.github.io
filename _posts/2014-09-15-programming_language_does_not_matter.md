---
title: Programming language does not matter
date: 2014-09-15
categories: [Architecture, PHP]
---

A few days ago I participated in quick presentation of significant e-commerce platform. 
The custom platform implemented mostly in PHP and designed as scalable and distributed system. 
And I was really impressed! Below you can find a short summary of chosen libraries, frameworks and tools. 

*Symfony* - The leading PHP framework to create web applications. 
Very similar to Spring Framework, you will get dependency injection, layered architecture and good support for automated testing.

*Doctrine* - Object to relational mapper (ORM), part of the Symfony framework.  Very similar to JPA.

*Composer* - Dependency manager for PHP, very similar to NPM.
 
*Gearman* - Generic application framework to farm out work to other machines or processes. Somehow similar to YARN. 

*Varnish* - HTTP reverse proxy.

*Memcached* - Distributed key value cache.

*RabbitMQ* - Messaging middleware based on AMPQ protocol. Used for distributed services integration but also for decoupled request reply communication.  

*logstash* - Log manager with tons of plugins to almost everything. The monitoring is really crucial in distributed systems.

The programming language does not really matter if you need scalable, distributed, easy to maintain and enhance system. 
You can apply excellent design using PHP or produce big ball of mud in Java.
