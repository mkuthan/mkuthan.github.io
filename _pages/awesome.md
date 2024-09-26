---
title: "Awesome notes"
permalink: /awesome/
---

When I found an interesting article or resource, if something opened my eyes I documented it here.

* [Big Data is dead](https://motherduck.com/blog/big-data-is-dead/) - by Jordan Tigani, ex googler, DuckDB advocate.

* [Apache Druid tuning guide](https://imply.io/developer/articles/learn-how-to-achieve-sub-second-responses-with-apache-druid/) - how to achieve sub-second responses.

* [Apache Beam Design Documents](https://s.apache.org/beam-design-docs) - These documents, managed by Googlers, provide an explanation of the Beam programming model and more.

* [Dataflow Streaming Execution Model](https://medium.com/google-cloud/streaming-engine-execution-model-1eb2eef69a8e) -
Processing within exactly-once guarantees and with latencies consistently in the low single-digit seconds.

* [How NAT traversal works](https://tailscale.com/blog/how-nat-traversal-works) -
How to traverse firewalls, SNAT, CGNAT in mesh networks.

* [Tracking health over debt](https://www.rea-group.com/about-us/news-and-insights/blog/what-good-software-looks-like-at-rea/) -
How to evaluate projects quality using lenses for: development, operations and architecture.

* [How to write efficient Flink SQL](https://www.alibabacloud.com/blog/how-to-write-simple-and-efficient-flink-sql_600148) -
What is a difference between: dual-stream, lookup, interval, temporal and window joins.

* [Part 1](https://engineering.atspotify.com/2023/04/spotifys-shift-to-a-fleet-first-mindset-part-1/),
[part 2](https://engineering.atspotify.com/2023/05/fleet-management-at-spotify-part-2-the-path-to-declarative-infrastructure/) and
[part 3](https://engineering.atspotify.com/2023/05/fleet-management-at-spotify-part-3-fleet-wide-refactoring/) -
Fleet-wide refactoring at Spotify, how to get 70% adoption of shared library in 7 days?

* [Lessons from Building Static Analysis Tools at Google](https://cacm.acm.org/magazines/2018/4/226371-lessons-from-building-static-analysis-tools-at-google/fulltext) -
Finding bugs is easy, but how make them actionable at scale?

* [BigQuery metadata](http://vldb.org/pvldb/vol14/p3083-edara.pdf) -
When Metadata is Big Data.

* [Prompt engineering](https://www.promptopedia.pl) -
How to talk to GenAI (in polish)?

* [Datadog incident](https://www.datadoghq.com/blog/2023-03-08-multiregion-infrastructure-connectivity-issue/) -
Multi-region and multi-cloud architecture didn't necessarily help to avoid the outage if you forget about legacy security in-place update channel.

* [The Engineer/Manager Pendulum](https://www.infoq.com/presentations/hands-on-coding-managers/) -
You don’t have to choose to be one or the other, but that you must choose one at a time.

* [JavaScript event loop](https://developer.mozilla.org/en-US/docs/Web/JavaScript/EventLoop)
and [Node.js event loop](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick#what-is-the-event-loop) -
Runtime models for JavaScript in the browser and server side.

* [Learn You a Haskell for Great Good!](http://learnyouahaskell.com/chapters) and [Learning Scalaz](https://eed3si9n.com/learning-scalaz/) -
Side by side comparision.

* [Valhalla](https://openjdk.org/projects/valhalla/design-notes/state-of-valhalla/01-background) -
Mechanical sympathy for Java.

* [test && commit || revert](https://medium.com/@kentbeck_7670/test-commit-revert-870bbd756864) -
How to reduce size of the change to increase velocity? Revert the change always if tests fail!

* [Learning different technologies based in the Scala Programming Language](https://www.scala-exercises.org) -
Coding exercises for Standard library, FP in Scala, Cats and other libraries.

* [Abstract Type Members versus Generic Type Parameters in Scala](https://www.artima.com/weblogs/viewpost.jsp?thread=270195) -
`trait Collection[T] {}` vs. `trait Collection { type T }`.

* [The Principles of the Flix Programming Language](https://flix.dev/principles/) -
Design principles of Flix programming language and Flix compiler. Awesome checklist.

* [Local-first applications with automerge](https://youtu.be/I4aVMYhL8Pk) -
Conflict-Free Replicated Data Type (CRDT), binary protocol, lossless snapshots, JavaScript and Rust (WebAssembly) implementations.

* [Re-architecting Apache Kafka for the cloud](https://youtu.be/ZSuoLgNWBRU) -
Tiered storage, Kraft cluster management (instead of ZK), multi-tenancy (consumer/producer quota, threads, connections, replication quota).

* [Understanding the Python GIL](https://youtu.be/Obt-vMVdM8s) -
Global interpreter lock, the reason why Python doesn't scale on multi-core architectures.

* [Roblox Return to Service 10/28-10/31 2021](https://blog.roblox.com/2022/01/roblox-return-to-service-10-28-10-31-2021/) -
Postmortem from non-trivial 73h outage due to Consul failure, why it took so long?
Consul monitoring didn't work without Consul 😜

* [Exception Handling Considered Harmful](https://www.lighterra.com/papers/exceptionsharmful/) -
If you have always perceived that exception handling doesn't really work.

* [The Scheduler Saga](https://youtu.be/YHRO5WQGh0k) -
Implement Goroutines from scratch: global run-queue, distributed run-queue, work stealing, handoff, cooperative preemption, FIFO vs LIFO.

* [A Guide to the Go Garbage Collector](https://go.dev/doc/gc-guide) -
Interesting how it differs from JVM, one or two knobs for configuration.

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
Don't use Git pre-commit hook for checking code quality, valuable checks take too much time to run.

* [Don’t deploy applications with Terraform](https://medium.com/google-cloud/dont-deploy-applications-with-terraform-2f4508a45987) -
Terraform is for static resources management, not for application deployment.
