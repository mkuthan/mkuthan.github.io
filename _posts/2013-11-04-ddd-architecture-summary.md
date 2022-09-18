---
title: "DDD Architecture Summary"
date: 2013-11-04
tags: [Architecture, DDD, Spring]
---

In this blog post you can find my general rules for implementing system using _Domain Driven Design_. Do not use them 
blindly but it's good starting point for DDD practitioners.

## <a name="bc"></a>Bounded Context

* Separate bounded context for each important module of the application (important from business partner perspective).
* Independent of each other (if feasible).
* For monolithic application separate _Spring Framework_ context for each bounded context, e.g: `applicationContext-domain-crm.xml`,
`applicationContext-domain-shipping.xml`, etc.
* CRUD like bounded contexts (user management, dictionaries, etc.) should be implemented as _Anemic Domain Model_.

## <a name="domain"></a>Domain

* Place for application business logic.
* Must be independent of the technical complexity, move technical complexity into [infrastructure](#infrastructure).
* Must be independent of the particular presentation technology, move presentation related stuff into [web](#web).
* Internal package structure must reflect business concepts ([bounded contexts](#bc)), e.g: `crm`, `shipping`, `sales`,
`shared`, etc.

## <a name="dm"></a> Domain Model

* Rich model, place for: entities, domain services, factories, strategies, specifications, etc.
* Best object oriented practices applied (SOLID, GRASP).
* Unit tested heavily (with mocks in the last resort).
* Unit tests executed concurrently (on method or class level).
* Meaningful names for domain services e.g: `RebateCalculator`, `PermissionChecker`, not `RebateManager` or 
`SecurityService`.
* Domain services dependencies are injected by constructor.
* Having more than 2~3 dependencies is suspicious.
* Entities aren't managed by containers.
* Aggregate root entities are domain events publishers (events collectors).
* Aggregates in single bounded context might be strongly referenced (navigation across objects tree).
* Aggregates from different bounded contexts are referenced by business keys (if feasible).
* No security, no transactions, no aspects, no magic, only plain old Java.
* Interfaces for domain services when the service is provided by [infrastructure](#infrastructure).
* No interfaces for domain services implemented in the domain model itself.

## <a name="as"></a>Application Services

* Orchestrator and facade for actors under Model.
* Place for security handling.
* Place for transactions handling.
* Must not deliver any business logic, move business logic into [domain model](#dm). Almost no conditionals and loops.
* Implemented as transactional script.
* No unit tests.
* Acceptance tests executed against this layer.
* Cglib proxied, proxy must be serialized by session scoped beans in [web](#web) layer.
* Dependencies are injected on field level (private fields).
* Ten or more dependencies for single application service isn't a problem.
* Application services are also domain event listeners.
* Always stateless.
* No interfaces, just implementation.

## <a name="ab"></a>Application Bootstrap

* Initial application data.
* Loaded during application startup (fired by `BootstrapEvent`) if application storage is empty.
* Loading order is defined with Spring `Ordered` interface.
* Data is loaded within Model API.
* Data might be loaded within [application services](#as), e.g: load sample Excel when application is integrated with 
external world this way.
* No tests, bootstrap is tested during application startup on daily basis.

## <a name="infrastructure"></a>Infrastructure

* Place for technical services
* Must not deliver any business logic, move business logic into [domain](#domain).
* Internal package structure must reflect technical concepts, e.g: `~infrastructure.jpa`, `~infrastructure.jms`, 
`~infrastructure.jsf`, `~infrastructure.freemarker`, `~infrastructure.jackson`, etc.
* Shared for all bounded context of the application. For more complex applications, separate technical services e.g:
`~infrastructure.jpa.crm`, `~infrastructure.jpa.shipping`, etc.
* Class names must reflect technical concepts, e.g.: `JpaCustomerRepository`, `JaksonJsonSerializer`, 
not `CustomerRepositoryImpl`, `JsonSerializerImpl`.
* Integration tested heavily (with _Spring Framework_ context loaded).
* Integration tests executed by single thread.
* Test execution separated from unit tests within test groups.
* Separate _Spring Framework_ context for each technical concept, e.g: `applicationContext-infrastructure-jpa.xml`, 
`applicationContext-infrastructure-jms.xml`, etc.
* Separate and independent Spring test context for each technical module, e.g: `testContext-jpa.xml`, 
`testContext-jms.xml`, etc.

## <a name="web"></a>Web

* Client specific facade (REST, MVC, JSF, etc.)
* Place for UI logic (not applicable for JavaScript client and REST)
* Delegates requests to [application services](#as)
* No transactions, no method level security, move security and transactions to [application services](#as).
* No business logic, move business logic into [domain](#domain).
* Tested with mocked application services.
* Tested with loaded spring context for MVC controllers (if applicable).
* Serializable session scoped beans (to be safe all beans in this module should be `java.io.Serializable`).
* Internal package structure must reflect UI organization structure, it might be similar to project _sitemap_.
* Top level package might reflect technology or architecture e.g: `presentation`, `rest`, `mvc`, `jsf`, etc. 
