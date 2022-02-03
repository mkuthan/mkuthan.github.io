---
title: Rrelease It! - book review
date: 2014-06-21
categories: [Books, Craftsmanship]
---

Recently I read excellent book _Release It!_ written by Michael Nygard. 
The book is 7 years old and I don't know how I could miss the book until now.   

Michael Nygard shows how to design and architect medium or large scale web applications. 
Real lessons learnt from the trenches not golden rules from ivory architects.

This blog post is a dump of taken notes when I was reading the book. 
The list could be used as a checklist for system architects and developers.
There is no particular order of the notes, perhaps there are duplications too.

* _admin access_ - should use separate networks than regular traffic, if not administrator will not be able connect to the system when something is wrong.

* _network timeouts_ - should be always defined, if not our system could hang if there is a problem with remote service.

* _firewall_ - be aware of timeouts on firewall connection tracking tables, if the connection is unused for long time (e.g connection from the pool), firewall could drop packets silently.

* _failure probability_ - are dependant, not like during coin toss.
 
* _3rd party vendors_ - their client library often sucks, you can not define timeouts, you can not configure threading correctly. 

* _method wait_ - always provide the timeout, do not use method ``Object.wait()``.

* _massive email with deep links_ - do not send massive emails with deep links, bunch of requests to single resource could kill your application.
 
* _threads ratio_ - check front-end and back-end threads ratio, the system is as fast as its slowest part.

* _SLA_ - define different SLAs for different subsystems, not everything must have 99.99%

* _high CPU utilization_ - check GC logs first.

* _JVM crash_ - typical after OOM, when native code is trying to allocate memory - ``malloc()`` returns error but only few programmers handle this error.

* _Collection size_ - do not use unbounded collections, huge data set kills your application eventually.

* _Outgoing communication_ - define timeouts.

* _Incoming communication_ - fail fast, be pleasant for other systems.

* _separate threads pool_ - for admin access, your last way to fix the system.

* _input validation_ - fail fast, use JS validation even if validation must be duplicated.

* _circuit braker_ - design pattern for handling unavailable remote services.

* _handshake in protocol_ - alternative for _circuit braker_ if you desing your own protocol.

* _test harness_ - test using production like environment (but how to do that???)

* _capacity_ - always multiply by number of users, requests, etc.

* _safety limits on everything_ - nice general rule.

* _oracle and connection pool_ - Oracle in default configuration spawns separate process for every connection, check how much memory is used only for handling client connections.

* _unbalanced resources_ - underestimated part will fail first, and it could hang whole system.

* _JSP and GC_ - be aware of ``noclassgc`` JVM option, compiled JSP files use perm gen space.

* _http sessions_ - users do not understand the concept, do not keep shopping card in the session :-)

* _whitespaces_ - remove any unnecessary whitespace from the pages, in large scale it saves a lot of traffic.

* _avoid hand crafted SQLs_ - hard to predict the outcome, and hard to optimize for performance.

* _database tests_ - use the real data volume.

* _unicast_ - could be used for up to ~10 servers, for bigger cluster use multicast.

* _cache_ - always limit cache size.

* _hit ratio_ - always monitor cache hit ratio.

* _precompute html_ - huge server resource saver, not everything changes on every request.

* _JVM tuning_ - is application release specific, on every release memory utilization could be different.

* _multihomed servers_ - on production network topology is much more complex.

* _bonding_ - single network configured with multiple network cards and multiple switch ports.

* _backup_ - use separate network, backup always consumes your whole bandwidth.

* _virtual IP_ - always configure virtual IP, your configuration will be much more flexible.

* _technical accounts_ - do not share accounts between services, it would be security flaws.

+ _cluster configuration verification_ - periodically check configuration on the cluster nodes, even if the configuration is deployed automatically.

* _separate configuration specific for the single cluster node_ - keep node specific configuration separated from shared configuration.

* _configuration property names_ - based on function not nature (e.g: hostname is too generic).

* _graceful shutdown_ - do not terminate existing business transations.

* _thread dumps_ - prepare scripts for that, during accident time is really precious (SLAs).
 
* _recovery oriented computing_ - be prepared for restarting only part of the system, restarting everything is time consuming.

* _transparency_ - be able to monitor everything.

* _monitoring policy, alerts_ - should not be defined by the service, configure the policies outside (perhaps in central place).

* _log format_ - should be human readable, humans are the best in pattern matching, use tabulators and fixed width columns.

* _CIM_ - _SNMP_ superior.

* _SSL accelerator_ - what it really is???

* _OpsDB monitoring_ - measurements and expectations, end to end business process monitoring.

* _Node Identifiers_ - assign to teams in block.

* _Observe, Orient, Decide, Act_ - military methodology, somehow similar to Agile :-)

* _review_ - tickets, stack traces in log files, volume of problems, data volumes, query statistics periodically.

* _DB migration_ - expansion phase for incompatible schema changes.
