---
title: SOA Patterns - book review
date: 2014-06-26
tags: [Books, Architecture]
---

## Overview

I took this book from my bookshelf when I was preparing internal presentation about micro services for my Roche colleagues.
I was mainly interested in _Saga_ and _Composite Front End_ patterns. But when I started, I decided to read rest of the book.

## Patterns

Below you can find my short summary about every pattern described in the book:

### Service Host

Every service needs the host where it works. For me _Spring Framework_ is excellent example of the service host.

### Active Service

Very similar to Micro Services concept, when the service should be autonomous.

### Transactional Service

I know a few alternative names of this pattern: Unit of Work, Open Session in View. In JEE world implemented using `ThreadLocal`.

### Workflodize

Strange pattern name. I don't really like complexity of workflow engines and prefer simple object oriented finite state machine implementation. 

### Edge Component

Separate infrastructure code from domain. Just simple like that.

### Decoupled Invocation

Use event / command bus for communication. 

### Parallel Pipelines

Apply Unix philosophy to your services. SRP on the higher level.

### Gridable Service

Horizontal scaling.

### Service Instance

Horizonatal scaling. 

### Virtual Endpoint

Make your deployment configuration flexible. 

### Service Watchdog

Service monitoring should be built-in.

### Secured Message

Encrypt what should be secured on the message level (privacy, integrity, impersonation).

### Secured Infrastructure

Encrypt what should be secured on the protocol level (privacy, integrity, impersonation).

### Service Firewall 

Security on the network level. Expose only what is really needed.

### Identity Provider

Single Sign On.

### Service Monitor

Monitoring on the business process level.

### Request/Reply 

Synchronous point to point communication.

### Request/Reaction

Asynchronous point to point communication.

### Inversion of Communications

Command Bus, Event Bus, messaging middleware in general. Complex Event Processing (CEP).

### Saga

Long running business transactions. Distributed transactions without XA.

### Reservation

Related to Saga, how to avoid XA transactions.

### Composite Front End

How to compose services into single web application? Author doesn't answer my doubts in this chapter.

### Client/Server/Service

How to deal with legacy systems. How to move from monolithic architecture to SOA.

### Service Bus

Message Bus, Service Bus, ESB - nice explanation.

### Orchestration

Externalize business long running processes. But still encapsulate business logic in services not in the orchestrator!

### Aggregated Reporting

Looks like CQRS for me.

## Antipatterns

Funny names for real problems when SOA is used:

* Knot - problems with coupling.
* Nanoservice - problems with bounded contexts.
* Transactional Integration - problems with XA transations.
* Same Old Way - problems with CRUD like services.

## Summary

For sure it's worth reading but I expected more from Arnon Rotem-Gal-Oz. 
Sometimes I felt that author covers only the top of the iceberg, when demons are under the hood.
The sample code fragments aren't very helpful, with high accidental complexity but do not clearly show the problem. 


In addition the book was published in 2012 but you will easily realized that author had started ten years before, some parts seems to be outdated.