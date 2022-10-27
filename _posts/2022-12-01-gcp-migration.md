---
title: "Migration to Google Cloud Platform"
date: 2022-12-01
categories: [Books]
tags: [Architecture]
---

Google Cloud Platform (GCP)

## IT procurement

> Cloud platforms contain hundreds of individual products, making a comparison rather meaningless.
> Better approach is to compare your company's vision with the provider's product strategy.

Why Allegro migrated on-premise big data pipelines to GCP?

* Machine learning (ML) for everyone
* Scalability
* No scoring or cloud matrix comparison
* TODO

> In the case of cloud, you're not looking to replace an existing component that serves a specific organizational need.
> Rather the opposite, you're looking for a product that enables your organization to work in fundamentally different way.

Allegro adjusted it's operating model to Google Cloud Platform.
On-premise Hadoop cluster hasn't been replaced by something similar in cloud.

* BigQuery isn't a Hive / HDFS / Parquet replacement, it enables TODO
* Dataproc / ephemeral clusters / Spark on K8S / etc.
* Dataflow - unified batch and streaming
* Vertex AI - TODO: ask Maciek how it has change the way they're working
* TODO

> Traditional IT budgets are set up at least a year in advance, making the predictability a key consideration.
> IT procurement tends to negotiate multiyear license terms for the software they buy.

* Cloud elastic pricing turns Allegro procurement upside down.
* TODO

## Operations

> In traditional IT, security, hardware provisioning and cost control are the operations teams' responsibility, whereas features delivery and usability are in the application teams' court.

* In Allegro development teams have operated their software for years.
* Operations teams develops only internal PaaS like K8S od Hadoop clusters.

> Te cloud increases transparency, allowing you to monitor usage and policy violations rather than restricting processes upfront.

* TODO: processes in Allegro

## Cloud thinks in the first derivative

> Digital companies aren't looking to optimize existing processes -- they chart new territory by looking for new models.

* Allegro has always be a digital e-commerce company.
* Migration is a chance for Allegro to widely adopt data mesh and machine learning.
Every development team in Allegro should be able to play with data.

> Instead of being a good guesser (economy of scale), companies need to be fast learners (economy of speed)

In Allegro it depends on the team or area.
Some of them plan everything upfront for the next quarters, build detailed roadmaps, and don't like unplanned changes.
Others don't plan a lot, but self organize for experimentation and to quickly adopt changes.

## TODO

> Increasing your run budget may be a good thing

TODO: it might be a sign of business success, for example paying double for twice as many users and twice as many benefit

> You shouldn't run software that you didn't build

TODO

> If something breaks, you're guilty until proven innocent

TODO

> I attempted a couple of small migrations with highly motivated teams but without a program manager. They achieved their objectives, but with participant burnout and huge cost and time overruns.

TODO

> Automation and federation to achieve even faster migration velocity.

TODO

> Your metrics will evolve over time along with your migration, transitioning from technical metrics to business relevant value-oriented metrics.

TODO

> For most enterprises, a hybrid-cloud scenario is inevitable (...) have a good plan for how to make the best use of it.
> Five multi-cloud scenarios: arbitrary, segmented, choice, parallel, portable.

TODO

> If you want to achieve true hybrid-cloud setup (...) uniform management and deployment across the environments.

TODO

> Most failure scenarios break the abstraction.

TODO

> Excessive complexity is nature's punishment for organizations that are unable to make decisions.

TODO

> The most subtle and most dangerous type of lock-in is the one that affects your thinking.

TODO

> Technology Radar designated "Generic Cloud Usage" as Hold already in late 2018.

TODO

> Cloud and automation add a new "ility": disposability.

TODO

> Docker was designed and built for developers, not operations staff.

TODO

> Five characteristic of Cloud-Ready applications: Frugal, Relocatable, Observable, Seamlessly updatable, internally Secured, failure Tolerant (FROSST).

TODO

> A cloud journey isn't a one-time shot, but an ongoing series of learning and optimization.

TODO

> In the cloud, your biggest levers are sizing and resource lifetime.

`Costs [$] = size[units] * time[hours] * unit costs[$/unit/hour]`

What about DEV/TEST?

TODO

> Exchange rate

TODO

> The most expensive server is the one that’s not doing anything. Even at a 30% discount.

TODO

> Shifting from robustness and redundancy to resilience and automation is one of several ways how the cloud can defy existing contradictions, such as providing better uptime at lower cost.

TODO

> You might be surprised that changing providers wasn't on the list of major savings vehicles.

TODO

> If your data is largely static, you can save a lot in pay-per-query model by creating interim results tables.

TODO

> Knowing your needs and usage patterns is the most important ingredient into reducing costs.

TODO

> When comparing prices, don't expect huge difference in pricing between providers -- open market competition is doing its job just fine.

TODO

> Discount negotiations

TODO

> Lowering costs but making them visible in the process can be perceived as "costing more" by your internal customers.

TODO

> Deprecation allocates capital expenses spread over a period of time.

TODO

> Monitoring cost should be a standard element of any cloud operation, just like monitoring for traffic and memory usage.

TODO

> Unfortunately, many enterprise cloud deployments detach developers from billing and payments.

TODO

> You want to encourage developers to build dynamic solutions that auto-scale, so the chances that you’re going to see some surprises on your bill are high enough that it’s wise to just allocate that money up front.

Tuition - special money for learning from mistakes.

TODO

> Overreacting isn't a good idea (...) the cloud is there to give developers more freedom and more control.

TODO

> Use free trials to try things instead of overcommitting.
