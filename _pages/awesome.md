---
title: "My awesome notes"
permalink: /awesome/
---

When I found an interesting article or resource, if something opened my eyes I documented it here.

* [Understanding the Python GIL](https://youtu.be/Obt-vMVdM8s) -
  Global interpreter lock, the reason why Python doesn't scale on multicore architectures.

* [Roblox Return to Service 10/28-10/31 2021](https://blog.roblox.com/2022/01/roblox-return-to-service-10-28-10-31-2021/) -
  Postmortem from non-trivial 73h outage due to Consul failure, why it took so long? 
  Consul monitoring did not work without Consul 😜

* [Exception Handling Considered Harmful](https://www.lighterra.com/papers/exceptionsharmful/) -
  If you have always perceived that exception handling doesn't really work.

* [The Scheduler Saga](https://youtu.be/YHRO5WQGh0k) -
  Implement Goroutines from scratch: global run-queue, distributed run-queue, work stealing, handoff, cooperative preemption, FIFO vs LIFO.

* [A Guide to the Go Garbage Collector](https://go.dev/doc/gc-guide) -
  Interesting how it differs from JMV, one or two knobs for configuration.

* [Minimal version selection (MVS)](https://research.swtch.com/vgo-mvs) -
  Dependency management algorithm to produce high fidelity builds.

* [Documenting architecture decisions](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions) -
  One or two pagers with the motivations behind important architecture decisions, without ADRs this kind of knowledge evaporates.

* [Code review automated comments](https://github.com/reviewdog/reviewdog) -
  Framework used by many tools for commenting code reviews (documentation linters, static code analysis, test coverage).

* [Write a documentation like a pro](https://vale.sh) -
  Apply Microsoft or Google [style guide](https://github.com/errata-ai/packages) for your writing.

* [Test sizes](https://testing.googleblog.com/2010/12/test-sizes.html) -
  Google way for tests classification: small, medium, large (in addition to test scopes: unit, integration, e2e).

* [Pre-commit: Don’t Git hooked!](https://www.thoughtworks.com/insights/blog/pre-commit-don-t-git-hooked) -
  Do not use Git pre-commit hook for checking code quality, valuable checks take too much time to run.

* [Don’t deploy applications with Terraform](https://medium.com/google-cloud/dont-deploy-applications-with-terraform-2f4508a45987) -
  Terraform is for static resources management, not for application deployment.
