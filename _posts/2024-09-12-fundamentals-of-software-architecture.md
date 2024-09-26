---
title: "Fundamentals of software architecture -- book review"
date: 2024-09-12
tags: [Software Engineering, Architecture]
categories: [Books]
---

Recently I've read [Fundamentals of Software Architecture](https://www.oreilly.com/library/view/fundamentals-of-software/9781492043447/)
written by [Mark Richards](https://www.linkedin.com/in/markrichards3/)
and [Neal Ford](https://nealford.com).
I found this book valuable, even though my company doesn't have a formal architect role.
At Allegro, the most experienced senior software engineers take on the responsibilities of a software architect in addition to their regular development duties.

![Fundamentals of Software Architecture](/assets/images/2024-09-12-fundamentals-of-software-architecture/bookcover.jpg)

At the end of the book, there is a self-assessment section, which I partially summarized by writing my answers in this blog post.

## What are the four dimensions that define software architecture

1. Architecture Characteristics: These define the success criteria of a system, such as performance, scalability, and security. They're orthogonal to the system's functionality.
2. Structure: This refers to the type of architecture style or styles used in the system, such as microservices, layered, or microkernel architectures.
3. Architecture Decisions: These are the rules and guidelines that dictate how a system should be constructed. They include choices about technologies, frameworks, and design patterns.
4. Design Principles: These are guidelines that help development teams make decisions about how to implement the system. They're not hard-and-fast rules but rather best practices to follow.

## What's the difference between an architecture decision and a design principle

Architecture decisions are specific choices that define the architecture, while design principles are broader guidelines that influence those choices.

## List the eight core expectations of a software architect

1. Make architecture decisions
2. Continually analyze the architecture
3. Keep current with latest trends
4. Ensure compliance with decisions
5. Diverse exposure and experience
6. Have business domain knowledge
7. Possess interpersonal skills
8. Understand and navigate politics

## What's the first law of software architecture

Everything is a tradeoff

## List the three levels of knowledge in the knowledge triangle

1. Stuff you know
2. Stuff you know you don't know
3. Stuff you don't know you don't know

## What are the ways of remaining hands-on as an architect

* Coding regularly
* Side projects
* Pair programming
* Technical reading
* Training and courses
* Code reviews
* Prototyping and experimentation
* Mentoring

## What's meant by the term connascence

Describe the degree to which different parts of a system are interdependent

## What's the difference between static and dynamic connascence

* Static connascence occurs at the source code level, and developers can identify it by examining the code itself.
* Dynamic connascence is related to the runtime behavior of the system and developers can only identify it during execution, such as the order of calls.

## What's the strongest form of connascence

Connascence of Identity: This occurs when many components must reference the same entity, meaning any change to the identity of that entity requires changes across all components that reference it.
For example, if the format or structure of the entity identifier is changed from a numeric to an alphanumeric code, you need to update every module that references this entity.

## What's the weakest form of connascence

Connascence of Name, this occurs when many components must agree on the name of an entity. For example, if the name of a method changes, all references to that method must also be updated.
Usually straightforward to manage and refactor with IDE.

## Which is preferred static or dynamic connascence

Static connascence refers to dependencies that the compiler can check at compile time, such as type checking and method signatures.
Dynamic connascence, on the other hand, involves dependencies that are only checked at runtime, such as dynamic method calls or reflection.

In a code base, it's preferable to use static connascence over dynamic connascence.

## Give an example of an operational characteristic

* Availability
* Continuity (disaster recovery capability)
* Performance
* Recoverability
* Reliability/safety
* Robustness
* Scalability

## Give an example of a structural characteristic

* Configurability
* Extensibility
* Installability
* Leverageability (ability to reuse common components)
* Localization
* Maintainability
* Portability
* Upgradeability

## Give an example of a cross-cutting characteristic

* Accessibility
* Archivability (will the data need to be archived or deleted after a period of time)
* Authentication
* Authorization
* Legal
* Privacy
* Security
* Supportability
* Usability/achievability (level of training required for users to achieve their goals with the application)

## Why it's a good practice to limit the number of characteristics an architecture should support

* Avoiding complexity: Keep the architecture simpler and more understandable.
* Resource allocation: Ensure that resources are effectively allocated to the most critical aspects of the system.
* Trade-off management: Ensure that the system meets its most important goals without being overburdened by conflicting requirements.

## What's an architectural quantum

It refers to the smallest unit of an architecture that can be independently deployed and tested, encompassing all the necessary components to fulfill a specific business function.

## What's the difference between technical partitioning and domain partitioning

Technical partitioning focuses on technical roles, while domain partitioning focuses on business functionality. Domain partitioning often makes it easier to manage changes in business requirements, whereas technical partitioning can complicate changes due to inter-layer dependencies.

## Under what circumstances would technical partitioning be a better choice over domain partitioning

* Small or simple applications
* Homogeneous teams, for example: front-end developers, back-end developers
* Legacy systems with technical partitioning
* Standardized processes, for example: compliance with certain regulations or industry standards
* Performance optimization

## List the eight fallacies of distributed computing

1. The network is reliable
2. Latency is zero
3. Bandwidth is infinite
4. The network is secure
5. Topology doesn’t change
6. There is one administrator
7. Transport cost is zero
8. The network is homogeneous

## What's stamp coupling

Stamp coupling, also known as data-structured coupling, occurs when modules share a composite data structure and use only parts of it.
This can lead to issues where changes in the unused parts of the data structure might affect the module that doesn’t need those parts.

## What's the difference between an open layer and a closed layer

An open layer permits requests to bypass it and directly access any layer below it.
A closed layer requires that all requests pass through it before reaching any lower layer.

## What's the architecture sinkhole anti-pattern

The architecture sinkhole anti-pattern occurs when requests pass through multiple layers of an architecture without any significant processing or logic being applied at each layer.
Essentially, the layers act as mere pass-throughs, adding unnecessary complexity and overhead without providing any real value.

## Name the four types of filters and their purpose in pipeline architecture

* Input filters: Transform raw data into a form that's suitable for later processing stages.
* Transform filters: Enrich the data or apply business logic.
* Output filter: Convert the data into the final format and write it to the destination.
* Error filters: Handle exceptions and errors that may occur during processing.

## In what way does the pipeline architecture support modularity

* Separation of Concerns: Each step in the pipeline has a specific task, such as handling input, performing transformations, managing output, or handling errors.
* Reusability: Steps can be designed as reusable modules that can be plugged into different pipelines.
* Scalability: Individual steps can optimize, replace, or replicate without affecting other parts of the pipeline.
* Testing and debugging: Each step allows independent testing and debugging, making it easier to identify and fix issues.

## What's domain/architecture isomorphism

Principle where the structure of the software architecture closely mirrors the structure of the problem domain it's designed to address.
For example, in an operating system, the micro-kernel architecture's minimal core will handle only the most fundamental aspects of the system (like process communication and basic I/O), while other services (like file systems and device drivers) will be handled by external modules.

## What's a primary difference between broker and mediator topologies

* In a broker topology, individual components or services communicate with each other through a message broker.
  Services are loosely coupled because they don't need to know each other’s location or protocol.
* In a mediator topology, a mediator orchestrates and manages the communication between services.
  This approach is suitable for complex interactions that require coordination, transformation, and aggregation of many services.

## What's a primary aspect fo space-based architecture that differentiates in from other architecture styles

It splits both the processing and the storage across many servers.
The Space Based Architecture pattern is designed to offer high scalability, fault-tolerance, and low-latency data access by storing data in memory across many nodes.

## Name the four components that make up the virtualized middleware within a space-based architecture

* Message grid: Manages input request and session information
* Data grid: Manage the data replication between processing units when data updates occur
* Processing grid: Manages distributed request processing when there are many processing units, each handling a part of the application
* Deployment manager: Manages the dynamic startup and shutdown of processing units based on load conditions

## What's a difference between replicated cache and distributed cache

Replicated cache:

* Same data replicated to all nodes
* High fault tolerance, all nodes have the same data
* Low latency for read operations because each node has a full copy of the data
* Useful when read operations significantly outweigh write operations

Distributed cache:

* Stores different pieces of data on different cache nodes
* Lower compared to replicated cache
* Can be higher for read operations, since data might not be present on the local node
* Suitable for large datasets and when read and write operations are balanced
