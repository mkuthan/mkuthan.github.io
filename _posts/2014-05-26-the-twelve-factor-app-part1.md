---
title: "The Twelve-Factor App - part 1"
date: 2014-05-26
categories: [Architecture]
---

During my studies about "Micro Services" I found comprehensive (but short) document about _Twelve-Factor App_ methodology
for building software-as-a-service applications. The orginal paper is published at [12factor.net](http://12factor.net/).
 
Below you can find a short summary of my experiences for the first part of the document.
There is also a [second part](http://mkuthan.github.io/blog/2014/05/27/the-twelve-factor-app-part2/) of this blog post series.

## 1. Codebase

> There is always a one-to-one correlation between the codebase and the app

I had the chance to use setup, where Subversion repository was shared for many projects. 
Only once, and I said - "never again".
I remember problems with release management, setting up access rights, and crazy revision numbers.

## 2. Dependencies

> A twelve-factor app never relies on implicit existence of system-wide packages

I remember a setup where you spent whole day to build all dependencies (a lot of C and C++ code). 
The solution was a repository with compiled and versioned dependencies. 
Almost everything was compiled statically with minimal dependency to the core system libraries like stdc.

Right now I build projects using Maven repositories and artifacts.
But it is not enough for twelve-factor app, and I fully agree.
My next step should be using "Infrastructure as a code" principle in practice.

## 3. Config

> strict separation of config from code

Some time ago my application was deployed on the wrong environment (WAR file prepared for QA environment was deployed on PROD).
It was one of the worst week in my career to rollback everything back. 
Never again, I fully agree that binary should be environment independent. Keep configuration out of binary artifact.

## 4. Backing Services 

> The code for a twelve-factor app makes no distinction between local and third party services

I do not fully understand this chapter. 
What I understood is that I should separate my domain from attached resources (local and third party services).
And it is what I have done many times:

* Externalize connection configuration
* Use Anti Corruption Layer between my domain and infrastructure (e.g: hexagonal architecture)
* Do not mix domain logic with infrastructure code.

## 5. Build, release, run

> The twelve-factor app uses strict separation between the build, release, and run stages

The difference between build and release stages is somehow new for me. 
My JEE applications are released and deployed to the Maven repository.
The deployed WAR files are deployable on any environment, the configuration is externalized and applied during Maven WAR overlay process.
The outcome of the overlay is not stored as a reference but maybe it should. The question is where to put release?
Again in the Maven repository or as a Bamboo build artifact?

What I apply during the _run stage_ is the database schema migration using _Liquibase_ or _Flyway_ and it really works.
I agree with author to keep this stage as small as possible.

And for sure direct changes on the production are prohibited. 
I had to clean up the project when the changes were not checked in to the repository once, never again.   
  
## 6. Processes

> Twelve-factor processes are stateless and share-nothing.

I have never used this concept but I agree with author. 
From the scalability perspective share-nothing architecture of stateless services is good.

Ok, 10 years ago I developed application using CORBA and there were remote calls to fully stateful remote object. 
Bad idea, really.
