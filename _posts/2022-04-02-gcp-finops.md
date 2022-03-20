---
title: "GCP FinOps for Data Pipelines"
date: 2022-04-02
categories: [GCP, FinOps]
tagline: "Streaming and batch use cases"
header:
    overlay_image: /assets/images/katie-harp-w45gZMWrJWc-unsplash.webp
    overlay_filter: 0.2
---

Do you monitor costs of the data pipelines in exactly the same way as you monitor the processing job health, latency or throughput?
Nowadays, taking care of costs efficiency is an integral part of every data engineer job.
I would like to share my own experiences with applying [FinOps](https://www.finops.org/introduction/what-is-finops/) discipline 
in organization within tens of data engineering teams and thousands of data pipelines.

## Overview

FinOps is a very broad term, but what is especially important from a data engineer perspective?

* You take ownership of the cloud usage and costs
* Cost is introduced as a regular metric of every data pipeline
* To optimize the costs you have to know detailed billing of every cloud resource  
* Cost analysis becomes a part of [definition of done](https://www.agilealliance.org/glossary/definition-of-done/)
* Premature costs optimization is the root of all evil (do you remember Donald Knuth paper about [performance](https://wiki.c2.com/?PrematureOptimization)?)

All my blog posts are written based on my own experiences, everything is battle tested and all presented screens are taken from the real systems.
And this time it will not be different, I will share how to keep costs of the streaming and batch pipelines on Google Cloud Platform under control.
On daily basis I manage the analytical data platform, the project with $100k+ monthly budget.
I'm also early Google Cloud Platform adopter and early FinOps practitioner in the organization, so
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
I do not use them on daily basis, so they are out of this blog post scope.

## Resource oriented costs tracking

Because Google Cloud Platform billings are oriented around projects, products and SKUs, the built-in cost reports are focused on projects, products and SKUs as well.
Below you can find the real example of the report for my test projects, the costs for the last 30 days grouped by product.
As you can see "Compute Engine" is a dominant factor in the billing.

![Billing dashboard](/assets/images/finops_billing_dashboard.webp)

Unfortunately, I must not enclose any financial details from production environments, my employer is a [listed company](https://www.google.com/finance/quote/ALE:WSE).
{: .notice--info}

The built-in [Cloud Billing Report](https://cloud.google.com/billing/docs/how-to/reports) page provides:

* Spending timeline with daily data granularity
* Basic forecasts at the end of the current billing period
* Change to the previous billing period  
* Ability to filter or split the costs by project, product or SKU
* Insight into credits, discounts and promotions

At first, it looks that the cloud billing report plots everything you need, doesn't it?
Imagine the project with a dozen of different data pipelines using the same type of resources.

* How do you know the total cost of every single pipeline? 
* Does the cost of all resources utilized by the pipeline "A" fit to its business case budget? 
* Billing report shows that 60% the monthly budget is utilized by SKU "X", which data pipeline should be optimized to lower the costs?
* Which resource causes the highest cost for the data pipeline?

As a data engineer I'm interested in the total costs of the selected data pipeline.
Then I need to have an ability to drill down into cost of every single resource used by this pipeline.
{: .notice--info}

## Data pipeline oriented costs tracking

How to get data pipeline oriented costs tracking in Google Cloud Platform?

1. Develop cloud resources [labelling convention](https://cloud.google.com/resource-manager/docs/creating-managing-labels#common-uses)
2. Apply the labels for ALL resources used by the data pipelines
3. Configure the [cloud billing data export](https://cloud.google.com/billing/docs/how-to/export-data-bigquery) to BigQuery
4. Craft Datastudio report aligned to the developed labelling convention, there is an [example](https://cloud.google.com/billing/docs/how-to/visualize-data)
5. Figure out the workarounds for GCP products for which 3) and 4) do not work

I would say that 2) and 5) are the toughest parts of the journey.

### Labeling convention

There is no single, suggested and well documented cloud resources labelling conventions to apply.
You have to develop your own methodology, but I would like to share a few best practices:

* Introduce labelling naming convention at the very beginning
* Prepare and share the documentation, it should be clear for any adopter how to apply and how to interpret the labels
* Add new labels only when they are really needed, too many labels do not help
* Continuously check the billing report for the resources without labels and fill the gaps
* Use the prefix for all labels to easily recognize your labels from the built-in ones

Below you can find the labelling convention I have created in my company:

* **allegro__sc_id** - Every Allegro library, application or data pipeline is registered in the internal service catalog. 
  This is a central repository of metadata like the service business or technical owners, responsible team, issue tracker etc.
  The perfect identifier for cloud resources labelling.
* **allegro__job_name** - Every data pipeline could consist of many processing jobs, so service identifier is not enough to identify every part of the pipeline.
  Job name is also more convenient to use than synthetic service identifier. 
  Many dashboards are organized around the job name.
* **allegro__branch_name** - When you develop and test new feature it is handy to assign the branch/feature name as a label.
  You will exactly know the cost of the experimentation phase.
  It is also quite useful during any technical upgrades, to verify that after change the job is still as cost-effective as before.
  Deploy the copy of the job aside and compare the costs to the current production.

I was also surprised when I realized that billing export does not provide any built-in labels for the well-known resources like BigQuery datasets, Pubsub topics or Cloud Storage buckets.
To mitigate this limitation the following additional labels are applied on the cloud resources:

* **allegro__topic_name** - The name of the Pubsub topic
* **allegro__dataset_name** - The name of the BigQuery dataset
* **allegro__bucket_name** - The name of the Cloud Storage bucket

If the data pipeline subscribes or publish to multiple Pubsub topics, produces multiple BigQuery datasets or writes to multiple Cloud Storage buckets 
the billing export will provide detailed costs for every single resource.

### Reports

Before I go into technical details I will present a few screens with the final reports I'm using on daily basis.
All reports are organized around the processing jobs, you can filter by job identifier, name or feature branch.
By default, the reports show cost for the last month within daily granularity, but you can always specify a different time range.
For every cloud product the most important SKUs are grouped and organized as "big numbers" to provide quick overview of the costs structure.
You can also filter by original product and SKU like on the built-in dashboards.

![Summary dashboard](/assets/images/finops_summary1_dashboard.webp)

TODO

![Summary dashboard without unlabelled costs](/assets/images/finops_summary2_dashboard.webp)

TODO

![Summary dashboard for single data pipeline](/assets/images/finops_summary3_dashboard.webp)


The summary page which shows the total costs of data pipelines, grouped by the job name.
You can easily compare the costs of different pipelines, it is often the best method to verify if the data pipeline is optimal or not.
If the job processes a given amount of data and costs ten times more than another similar job it is a first candidate for further inspection.
The report gives also a sneak peek of the job costs structure, you should look for savings where you spend the most.

![Dataflow dashboard](/assets/images/finops_dataflow_dashboard.webp)

The report crafted for Dataflow data pipelines. Dataflow is a fully managed ...
Pay special attention for Streaming and Shuffle services, 

TODO: Dataproc

![Dataprod dashboard](/assets/images/finops_dataproc_dashboard.webp)

TODO: BigQuery

![BigQuery dashboard](/assets/images/finops_bigquery_dashboard.webp)

TODO: Pubsub

![Pubsub dashboard](/assets/images/finops_pubsub_dashboard.webp)

TODO: Cloud Storage

![Pubsub dashboard](/assets/images/finops_storage_dashboard.webp)



-----------

## FinOps sources

Billing Export
BigQuery audit logs / information schema?

## Labelling convention

## Streaming use case

* pubsub topic
* pubsub subscriptions (regular + internal)
* dataflow
* logging

## Batch use case

* bigquery analysis
* bigquery storage api
* compute
* storage
* logging

## Scaling the discipline for the whole organization

## Summary

TODO
