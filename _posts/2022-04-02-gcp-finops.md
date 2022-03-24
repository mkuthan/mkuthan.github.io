---
title: "GCP FinOps for Data Pipelines"
date: 2022-04-02
categories: [GCP, FinOps]
tagline: "Streaming and batch use cases"
header:
    overlay_image: /assets/images/katie-harp-w45gZMWrJWc-unsplash.webp
    overlay_filter: 0.2
---

Do you monitor costs of the data pipelines in exactly the same way as you monitor the processing jobs' health, latency or throughput?
Nowadays, taking care of costs efficiency is an integral part of every data engineer job.
I would like to share my own experiences with applying [FinOps](https://www.finops.org/introduction/what-is-finops/) discipline 
in the organization within tens of data engineering teams and thousands of data pipelines.

## Overview

FinOps is a very broad term, but what is especially important from a data engineer perspective?

* You take ownership of the cloud usage and costs
* Cost is introduced as a regular metric of every data pipeline
* To optimize the costs you have to know detailed billing of every cloud resource  
* Cost analysis becomes a part of [definition of done](https://www.agilealliance.org/glossary/definition-of-done/)
* Premature costs optimization is the root of all evil (do you remember Donald Knuth paper about [performance](https://wiki.c2.com/?PrematureOptimization)?)

All my blog posts are based on my own practical experiences, everything is battle tested and all presented examples are taken from the real systems.
And this time it will not be different.
I will share how to keep costs of the streaming and batch pipelines on Google Cloud Platform under control.
On daily basis I manage the analytical data platform, the project with $100k+ monthly budget.
I'm also early FinOps practitioner in the organization, so
at the end of the post you will also learn how to scale the methodology across the teams and departments. 

## Google Cloud Platform resources

You may be surprised how complex is Google Cloud Platform billing.
There are many [products](https://cloud.google.com/products), every product has many [SKUs](https://cloud.google.com/skus), a hundred of different elements to track and count.
What are the most important cost factors in the Google Cloud Platform for data pipelines?

* Common resources
    - vCPU time
    - Memory
    - Persistent disks (standard or SSD)
    - Local disks (standard or SSD)
    - Network egress 
* Dataflow
    - Streaming service (for realtime pipelines)
    - Shuffle service (for batch pipelines)
* Dataproc
    - Licensing Fee
* BigQuery
    - Active, Long Term or Physical storage
    - Analysis
    - Streaming inserts
    - Storage API
* Pubsub
  - Message delivery basic
  - Inter-region data delivery
  - Topic/Subscription message backlog
* Cloud Storage
    - Standard, Nearline, Coldline or Archive storage
    - Class A operations
    - Class B operations
    - Data retrieval
    - Early deletion
* Cloud Composer
    - SQL vCPU time
    - Environment fee
* Monitoring & Logging
    - Metric Volume
    - Log Volume

There are even more products relevant to the data pipelines like [Vertex AI](https://cloud.google.com/vertex-ai),
[Bigtable](https://cloud.google.com/bigtable) (expensive beast), 
[Firestore](https://cloud.google.com/firestore) or 
[Memory Store](https://cloud.google.com/memorystore).

## Resource oriented costs tracking

Because Google Cloud Platform billings are oriented around projects, products and SKUs, the built-in cost reports are focused on projects, products and SKUs as well.
Below you can find the real example of the report for one of my test projects, the costs for the last 30 days grouped by the cloud product.
As you can see "Compute Engine" is a dominant factor in the billing.

![Billing dashboard](/assets/images/finops_billing_dashboard.webp)

Unfortunately, I must not enclose any financial details from the production environments because my employer is a [listed company](https://www.google.com/finance/quote/ALE:WSE).
If I had shown the real numbers I would have had severe troubles.
{: .notice--info}

As shown before, the built-in [Cloud Billing Report](https://cloud.google.com/billing/docs/how-to/reports) page provides:

* Spending timeline with daily data granularity
* Change since the previous billing period  
* Ability to filter or split the total costs by project, product or SKU
* Insight into discounts, promotions and negotiated savings

At first, it looks that the cloud billing report plots everything you need, doesn't it?
Unluckily it has many shortcomings. 
Imagine the project with a dozen of different data pipelines using the same type of resources.

* How do you know the cost of every single pipeline? 
* How to compare the costs of different data pipelines?
* Does the cost of all resources utilized by the pipeline "A" fit to its business case budget? 
* Billing report shows that most of the monthly budget is utilized by SKU "X", which data pipeline should be optimized to lower the costs?
* Which resource causes the highest cost for the given data pipeline?

As a data engineer I'm interested in the total costs of the selected data pipeline.
Then I need to have an ability to drill down into the cost of every single resource used by this pipeline.
{: .notice--info}

## Data pipeline oriented costs tracking

Now we have already defined the requirements for billing reports which helps us to get better costs understanding.
How to get data pipeline oriented costs tracking in Google Cloud Platform?

1. Develop cloud resources [labelling convention](https://cloud.google.com/resource-manager/docs/creating-managing-labels#common-uses)
2. Apply the labels for ALL resources used by the data pipelines
3. Configure the [cloud billing data export](https://cloud.google.com/billing/docs/how-to/export-data-bigquery) to BigQuery
4. Craft Datastudio report aligned to the developed labelling convention, there is an [example](https://cloud.google.com/billing/docs/how-to/visualize-data)
5. Figure out the workarounds for GCP products for which 3) and 4) do not work

I would say that 2) and 5) are the toughest parts of the journey.

### Labeling convention

There is no single, the best cloud resources labelling conventions to apply.
You have to develop your own methodology, but I would like to share a few best practices:

* Introduce labelling naming convention at the very beginning
* Prepare and share the documentation, it should be clear for any adopter how to apply and how to interpret the labels
* Add a new labels only when they are really needed, too many labels do not help
* Continuously check the billing report for the resources without labels and fill the gaps
* Use the prefix for all custom labels to easily recognize them from the built-in ones

Below you can find the labelling convention I have created in my company:

* **allegro__sc_id** - Every Allegro library, application or data pipeline is registered in the internal service catalog. 
  This is a central repository of metadata like service business/technical owners, responsible team, issue tracker etc.
  The perfect identifier for cloud resources labelling.
* **allegro__job_name** - Every data pipeline could consist of many processing jobs, so the service identifier is not enough to identify every part of the pipeline.
  Job name is also more convenient to use than synthetic service identifier. 
  Many dashboards are organized around the job name.
* **allegro__branch_name** - When you develop and test new feature it is handy to assign the branch/feature name as a label.
  You will exactly know the cost of the experimentation phase for this feature/branch.
  It is also quite useful during any technical upgrades, to verify that after change the job is still as cost-effective as before.
  Deploy the copy of the job aside the current production, and compare the costs.

I was also surprised when I realized that billing export does not provide any built-in labels for the well-known resources like BigQuery datasets, Pubsub topics or Cloud Storage buckets.
To mitigate this limitation the following additional labels should be applied on the cloud resources:

* **allegro__topic_name** - The name of the Pubsub topic, apply the label to every topic of subscription
* **allegro__dataset_name** - The name of the BigQuery dataset, apply the label to every dataset
* **allegro__bucket_name** - The name of the Cloud Storage bucket, apply the label to every bucket

If the data pipeline subscribes or publishes to multiple Pubsub topics, produces multiple BigQuery datasets or writes to multiple Cloud Storage buckets 
the billing export will provide detailed costs for every single resource.

### Summary reports

Before I start writing about technical details I will present a few screens with the final reports I'm using on daily basis.

![Summary dashboard with shared costs](/assets/images/finops_summary1_dashboard.webp)

The summary dashboard for given project (sc-9936-nga-dev) with calculated costs for the last 30 days.
The total spending of $3.9k with the major participation from Dataproc and Cloud Composer.
The most interesting part of the dashboard is a timeline with spending grouped by the jobs deployed in the project.
The majority of the costs in the development environment come from the shared infrastructure, and can not be assigned to any particular job.
In the production environment with the large dataset to process the situation is fully opposite.
The shared cost is only a small portion of the billing.

What if the shared infrastructure is filtered out for development environment to get more realistic view?

![Summary dashboard without shared costs](/assets/images/finops_summary2_dashboard.webp)


You will get the exact cost of every single data pipeline deployed in the project.
Easy to compare each other, easy to spot any anomaly or trends.

The next step is to filter the report by job identifier, name or feature branch.

![Summary dashboard for the single job](/assets/images/finops_summary3_dashboard.webp)

The job "event_v1" is a Dataflow data pipeline. It reads data from Pubsub, publishes results to Pubsub and also writes some data to BigQuery.
The costs structure is as follows:

* $61.8 for Dataflow, the highest share in the billing -- again it is a development environment with minimal amount of data. 
But the Dataflow streaming pipeline is deployed on 24/7 so the cost is relatively high even for small virtual machines.
* $4.7 for reading and publishing from/to Pubsub
* $0.7 for writing to BigQuery
* $0.1 tiny amount for Cloud Storage, perhaps for Dataflow staging bucket

### Dataflow report

![Dataflow dashboard](/assets/images/finops_dataflow_dashboard.webp)

### Dataproc report

![Dataprod dashboard](/assets/images/finops_dataproc_dashboard.webp)

### BigQuery report

![BigQuery dashboard](/assets/images/finops_bigquery_dashboard.webp)

### Pubsub report

![Pubsub dashboard](/assets/images/finops_pubsub_dashboard.webp)

### Cloud Storage report

![Pubsub dashboard](/assets/images/finops_storage_dashboard.webp)

## The challenges

TODO

### BigQuery Analysis

TODO

### BigQuery Storage API

TODO

### Dataflow internal subscriptions

TODO

## Scaling the discipline for the whole organization

TODO

## Summary

TODO
