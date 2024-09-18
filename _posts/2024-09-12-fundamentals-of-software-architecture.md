---
title: "Fundamentals of Software Architecture -- book review"
date: 2024-09-12
tags: [Software Engineering, Architecture]
categories: [Books]
---

Recently I've read [Fundamentals of Software Architecture](https://www.oreilly.com/library/view/fundamentals-of-software/9781492043447/)
written by [Mark Richards](https://www.linkedin.com/in/markrichards3/)
and [Neal Ford](https://nealford.com).
I found this book valuable, even though my company doesn't have a formal architect role.
At Allegro, the most experienced senior software engineers take on the responsibilities of a software architect in addition to their regular development duties.

![Fundamentals of Software Architecture ](/assets/images/2024-09-12-fundamentals-of-software-architecture/bookcover.jpg)

At the end of the book, there is a self-assessment section, which I summarized by writing my answers in this blog post.

## What are the four dimensions that define software architecture

1. **Architecture Characteristics**: These define the success criteria of a system, such as performance, scalability, and security. They're generally orthogonal to the system's functionality.
2. **Architecture Decisions**: These are the rules and guidelines that dictate how a system should be constructed. They include choices about technologies, frameworks, and design patterns.
3. **Structure**: This refers to the type of architecture style or styles used in the system, such as microservices, layered, or microkernel architectures.
4. **Design Principles**: These are guidelines that help development teams make decisions about how to implement the system. They're not hard-and-fast rules but rather best practices to follow.

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

* Coding Regularly
* Side Projects
* Pair Programming
* Technical Reading
* Training and Courses
* Code Reviews
* Prototyping and experimentation
* Mentoring

## What's meant by the term connascence

Describe the degree to which different parts of a system are interdependent

## What's the difference between static and dynamic connascence

* Static connascence occurs at the source code level and can be identified by examining the code itself
* Dynamic connascence is related to the runtime behavior of the system and can only be identified during execution, for example order of calls

## What's the strongest form of connascence

Connascence of Identity, this occurs when many components must reference the same entity, meaning any change to the identity of that entity requires changes across all components that reference it.

## What's the weakest form of connascence

Connascence of Name, this occurs when many components must agree on the name of an entity. For example, if the name of a method changes, all references to that method must also be updated.

## Which is preferred within a code base—static or dynamic connascence

Static connascence refers to dependencies that can be checked at compile time, such as type checking and method signatures.

Dynamic connascence, on the other hand, involves dependencies that are only checked at runtime, such as dynamic method calls or reflection.

Static connascence is preferred over dynamic connascence within a code base.

## Provide an example of an operational characteristic

Availability, Continuity (disaster recovery capability), Performance, Recoverability,
Reliability/safety, Robustness, Scalability

## Provide an example of a structural characteristic

Configurability, Extensibility, Installability, Leverageability/reuse (ability to reuse common components),
Localization, Maintainability, Portability, Upgradeability

## Provide an example of a cross-cutting characteristic

Accessibility, Archivability (Will the data need to be archived or deleted after a period of time),
Authentication, Authorization, Legal, Privacy, Security, Supportability,
Usability/achievability (Level of training required for users to achieve their goals with the application)

## Why it's a good practice to limit the number of characteristics an architecture should support

* Avoiding complexity - keep the architecture simpler and more understandable
* Resource allocation - ensures that these resources are effectively allocated to the most critical aspects of the system
* Trade-off management - ensure that the system meets its most important goals without being overburdened by conflicting requirements
