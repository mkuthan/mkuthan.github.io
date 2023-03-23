---
title: "Managing technical debt using Dependabot"
date: 2023-03-27
tags: [Software Engineering]
header:
    overlay_image: /assets/images/2023-03-27-managing-technical-debt/kelly-sikkema-JUPzi-F3Iww-unsplash.webp
    caption: "[Unsplash](https://unsplash.com/@kellysikkema)"
---

Today I would like to show you how to manage technical debt by updating project dependencies continuously.
Surprisingly, with this technique you will get much more goodies than just updated dependencies:

* Faster delivery of a new business features
* More predictable estimations
* Minimized risk of security vulnerabilities
* Lower risk of changes
* Frequent deployments
* Better end2end tests
* Less surprises from cloud infrastructure changes
* Software engineers satisfaction

## Software lifecycle

Technical debt tends to increase naturally over time.
The complexity grows, architecture doesn't match the new requirements, developers leave and come, priorities change and more.
I've observed the following enterprise software lifecycle:

* Initial phase, everyone is enthusiastic, team develops software using the latest and greatest tools and techniques
* Steady state phase, new business requirements come in a constant pace, experienced development team keep technical debt under control
* Maintenance phase, product is mature, almost no new business requirements, development team moves its focus to the new product

The oldest services and data pipelines your team supports, might work smoothly without any intervention.
You deployed them last time a quarter ago, you even didn't check it out from the repository to your newly installed laptop. You almost forget about these services, they're beyond the horizon.

Over time your small team has 20+ services and hundreds data pipelines to support, but only a few of them are under active development.
Everything seems to be fine until someone asks you for making an important and urgent change in one of the oldest services you support.

## Technical debt history

I'm primarily responsible for Allegro clickstream ingestion platform.
Highly scalable and fault-tolerant piece of services and data pipelines that process billions of events every day.
The platform is an important part of Allegro ecosystem, it delivers data for modeling recommendations, tuning search index, calculating ads metrics, building client profiles and more.

The oldest parts of the platform are 7--8 years old, stability is outstanding, everything just works.
Because a data model is like a generic map change requests are rare, data producers are able to change the payload without any change in the data pipelines.
Do you see a scratch on the glass?

Last year I read [Software Engineering at Google](/blog/2022/09/22/software-engineering-at-google/) book
and found interesting thesis:

> You may choose to not change things, but you need to be capable

Am I capable of making the ingestion platform change in finite time, expected quality and without a risk?
Quick analysis proved that components on the critical path are outdated and deployment intervals were longer than a quarter. For example:

* Java 8
* Scala 2.12
* Akka 2.15
* Kafka 2.6
* Avro 1.8

There were no end2end tests and deployment was a semi-manual process.
After merging features to the main branch, GitHub action released the components and put them into artifact registry.
Developers had to deploy released components using Allegro management GUI console on DEV, TEST, Canary and finally the PROD environment.
Not a big deal if you deploy components quarterly.

I convinced my engineering manager to update critical components in the clickstream ingestion platform.
We were going to spend precious resources not for delivering new features but to be capable of doing it 😀.
I was also thinking about how to prevent backsliding in the future with minimal effort.
If components rusted once, they will rust again and again.

## Automated dependency updates

What could be a reason for changes if there are no business change requests?

Dependency updates managed by Dependabot is a way to enforce updates for the projects in maintenance phase
{: .notice--info}

Enabling automated dependency updates is a 5 minute task for projects hosted on GitHub,
but preparing projects for such updates could take months.
You have to set up projects properly, if not - automated dependency updates will kill your productivity.
You have to apply changes and deploy often without any fear.

For the clickstream ingestion platform I created the list of minimal prerequisites to enable Dependabot:

* Fully automated end2end tests using Test Containers to be sure that changes won't introduce any regression
* Automated deployment on DEV, TEST and Canary environments to minimize manual tasks
* Automated checks on Canary environment using regular service metrics, on-duty engineers will get notification if canary environment doesn't work as expected

After the preparation step you can plan updates, one by one with a full deployment cycle every time.
We applied the following war plan for a battle with technical debt:

* Update Scala to version 2.13 to enable other dependency updates
* Update all dependencies to the latest versions compatible with Java 8
* Update Java to version 17, and carefully migrate GC flags to achieve at least the same performance like for Java 1.8
* Update a few remaining dependencies not compatible with Java 1.8
* With up-to-date runtime and dependencies migrate service from Mesos to Kubernetes cluster
* Enable Dependabot with weekly schedule to avoid backsliding

## How does it work

On almost every Monday we've got new pull requests with dependency updates.
They patiently wait in a queue until one of the software engineers has some spare time.
They review and merge Dependabot pull requests and back to the regular tasks.
If there are no incidents on the Canary environment, they promote deployment to the PROD environment.
It doesn't take more than half an hour.

If dependency update build fails, software engineers create an internal issue to analyse the problem.
It's always good to know that one of your dependencies isn't compatible any more.
If we can't afford the change in the near future, we document technical debt as an ignore clause in the Dependabot configuration file.

```yaml
updates:
  - package-ecosystem: "gradle"
    directory: "/"
    schedule:
      interval: "weekly"
    ignore:
      # Akka versions >= 2.7.x have changed licensing
      - dependency-name: "com.typesafe.akka:akka-*"
        versions: [ "[2.7.0,)" ]
```

If your corporate policy permits deployment without any acceptance, you could enable automatic pull request merge and deployments always if build is green.
In Allegro at least one software engineer must accept the change so I couldn't the process fully.

For services without risk of data loss, you should enable automated deployment on the PROD environment as well.
For example, if there is an incident for Kafka based data pipelines, deploy the job again and replay events from the Kafka cluster.

I would strongly recommend fighting with technical debt using Dependabot.
At least for the services on the critical path if they aren't developed actively.
It gives much more than updated dependencies.
Do you remember?

> You may choose to not change things, but you need to be capable
