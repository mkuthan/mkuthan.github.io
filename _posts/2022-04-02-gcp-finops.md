---
title: "GCP FinOps for data pipelines"
date: 2022-04-02
categories: [GCP, FinOps]
tagline: ""
header:
    overlay_image: /assets/images/katie-harp-w45gZMWrJWc-unsplash.webp
    overlay_filter: 0.2
---

Do you monitor costs of the data pipelines in exactly the same way as you monitor overall health, latency or throughput?
Nowadays, taking care of cost efficiency is an integral part of every data engineer job.
I would like to share my own experiences with applying [FinOps](https://www.finops.org/introduction/what-is-finops/) discipline 
in the organization within tens of data engineering teams and thousands of data pipelines.

All my blog posts are based on practical experiences, everything is battle tested and all presented examples are taken from the real systems.
And this time it will not be different.
On a daily basis I manage the analytical data platform, the project with $100k+ monthly budget.
I will share how to keep costs of the streaming and batch pipelines on Google Cloud Platform under control.

## FinOps overview

FinOps is a very broad term, but what is especially important from a data engineer perspective?

* You take ownership of the cloud resources usage and costs.
  With the great power of the public cloud comes great responsibility.
* Cost is introduced as a regular metric of every data pipeline.
  More data and more processing power cause higher costs.
* To optimize the costs you have to know detailed billing of every cloud resource.
  It does not make any sense to optimize the least expensive part of the system.
* Cost analysis becomes a part of [definition of done](https://www.agilealliance.org/glossary/definition-of-done/).
  After major changes you should also check the billings for any regression.
* Premature cost optimization is the root of all evil.
  Do you remember Donald Knuth's paper about [performance](https://wiki.c2.com/?PrematureOptimization)?
  
## Google Cloud Platform resources

You may be surprised how complex Google Cloud Platform billing is.
There are dozens of [products](https://cloud.google.com/products), every product has tens of [SKUs](https://cloud.google.com/skus), hundreds of different elements to track and count.
What are the most important cost factors in the Google Cloud Platform for data pipelines I'm familiar with?

* Common resources
  - vCPU
  - Memory
  - Persistent disks (standard or SSD)
  - Local disks (standard or SSD)
  - Network egress 
* Dataflow
  - Streaming service (for real-time pipelines)
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
  
There are other products relevant to the data pipelines, but unfortunately I do not use them on daily basis:

* [Vertex AI](https://cloud.google.com/vertex-ai)
* [Bigtable](https://cloud.google.com/bigtable)
* [Firestore](https://cloud.google.com/firestore)
* [Memory Store](https://cloud.google.com/memorystore)

## Resource oriented costs tracking

Because Google Cloud Platform billings are oriented around projects, products and SKUs, the built-in cost reports are focused on projects, products and SKUs as well.
Below you can find the real example of the report for my primary project, the costs for the last 30 days grouped by the cloud product.
As you can see "BigQuery" is a dominant factor in the billing.

![Billing dashboard](/assets/images/finops_cloud_billing_report.webp)

Unfortunately, I must not enclose any financial details from the production environments because my employer is a [listed company](https://www.google.com/finance/quote/ALE:WSE).
If I had shown the real numbers I would have had severe troubles.
{: .notice--warning}

The presented built-in [Cloud Billing Report](https://cloud.google.com/billing/docs/how-to/reports) page provides:

* Spending timeline within daily or monthly data granularity
* Change since the previous billing period 
* Ability to filter or split the total costs by projects, products or SKUs
* Insight into discounts, promotions and negotiated savings

At first, it looks like the cloud billing report plots everything you need, doesn't it?
Unluckily it has many shortcomings. 
Imagine a project with a dozen different data pipelines using the same type of cloud resources.

* How do you know the cost of every single pipeline? 
* How to compare the costs of different data pipelines?
* Does the cost of all resources utilized by the pipeline "A" fit to its business case budget? 
* Billing report shows that most of the monthly budget is utilized by SKU "X", which data pipeline should be optimized first?
* Which resource causes the highest cost for the given data pipeline?

As a data engineer I'm interested in the total costs of the selected data pipeline.
Then I need to have an ability to drill down into the cost of every single resource used by this pipeline.
{: .notice--info}

## Data pipeline oriented costs tracking

Now we have already defined the requirements for billing reports which helps us to get better costs understanding.
How to create data pipeline oriented costs tracking in Google Cloud Platform?

The recipe:

1. Develop cloud resources [labeling convention](https://cloud.google.com/resource-manager/docs/creating-managing-labels#common-uses)
2. Apply the labels for ALL resources used by the data pipelines
3. Configure the [cloud billing data export](https://cloud.google.com/billing/docs/how-to/export-data-bigquery) to BigQuery
4. Craft Datastudio report aligned with the developed labeling convention, there is an [example](https://cloud.google.com/billing/docs/how-to/visualize-data)
5. Figure out the workarounds for GCP products which do not provide necessary details in the billing data export

I would say that 2) and 5) are the toughest parts of the journey.

### Labeling convention

There is no silver bullet, the best cloud resources labeling conventions do not exist.
You have to develop your own labeling convention, but I would like to share a few best practices:

* Introduce labeling naming convention at the very beginning
* Prepare and share the documentation, it should be clear for any adopter how to apply and how to interpret the labels
* Add a new labels only when they are really needed, too many labels do not help
* Continuously check the billing report for the resources without labels and fill the gaps
* Last but not least, use the prefix for all custom labels to easily recognize them from the built-in ones

Below you can find the labeling convention I have created in my company:

* **allegro__sc_id** - Every Allegro library, application or data pipeline is registered in the internal service catalog. 
  This is a central repository of metadata like business/technical owners, responsible team, issue tracker etc.
  The perfect identifier for cloud resources labeling.
* **allegro__job_name** - Every data pipeline could consist of many jobs, so the service identifier is not enough to identify every part of the pipeline.
  Job names are also more convenient to use than synthetic service identifiers. 
  As you will see later, many dashboards are organized around the jobs names.
* **allegro__branch_name** - When you develop and test a new feature it is handy to assign the branch/feature name as a label.
  You will exactly know the cost of the experimentation phase for this feature/branch.
  It is also quite useful during major upgrades, to verify that after change the job is still as cost-effective as before.
  Deploy the copy of the job aside the current production, and compare the costs using the label.

I also realized very quickly that billing export does not provide any built-in labels for the well-known resources like BigQuery datasets, Pubsub topics or Cloud Storage buckets.
To mitigate this limitation the following additional labels are defined for the cloud resources used by the data pipelines:

* **allegro__topic_name** - The name of the Pubsub topic, apply the label to every topic of subscription
* **allegro__dataset_name** - The name of the BigQuery dataset, apply the label to every dataset
* **allegro__bucket_name** - The name of the Cloud Storage bucket, apply the label to every bucket

If the data pipelines subscribe or publish to multiple Pubsub topics, produce multiple BigQuery datasets or write to multiple Cloud Storage buckets 
the billing export will provide detailed costs for every single labeled resource.

### Overview report

Before I move to the technical details I will present a few screens with the reports I'm using on a daily basis.
In the end, applying the labels for all resources in the cloud is a huge effort, so it should pay off somehow.

![Overview dashboards](/assets/images/finops_overview1.webp)

The overview dashboard provides a short manual for newcomers and the total costs of all projects.
It also gives full transparency, you can easily compare the costs of every project in the company.
I've found that it is also a quite effective method to motivate the teams for doing the optimizations.
Nobody wants to be at the top.

The overview page is the landing page, at the very beginning data engineer should select the project he/she is interested in.
You can search for the project by name, and the list is ordered by spending.

![Overview dashboard - single project](/assets/images/finops_overview2.webp)

When you select the project, there is a second page of the report. 
The page with the project costs is **organized around data pipelines**.

![Project dashboard](/assets/images/finops_project.webp)

You can see the timeline with daily costs split by job name based on **allegro__job_name** label.
The most expensive named jobs in the project are: "event_raw", "clickstream-enrichment" and "meta_event".
The presented costs are the total costs of the jobs, all cloud resources used by the jobs are counted.
If the pipeline is deployed as Apache Spark job on the Dataproc cluster, it queries BigQuery and saves the computation results into Cloud Storage, the **costs of all used cloud resources are presented here**.

What does "null" mean here? 
Due to the historical reasons some labels are missing for the very first data pipelines migrated to GCP.
No labels, no detailed reports ...
{: .notice--info}

### Dataflow and Dataproc reports

In the presented project "sc-9366-nga-prod" there are two types of data pipelines:

* [Dataflow](https://cloud.google.com/dataflow) / [Apache Beam](https://beam.apache.org) for the real-time jobs
* [Dataproc](https://cloud.google.com/dataproc) / [Apache Spark](https://spark.apache.org) for the batch jobs

Let start with the next report crafted for Dataflow pipelines:

![Dataflow dashboard](/assets/images/finops_dataflow.webp)

Again, the timeline is organized around the data pipelines. 
But now, the presented costs come from the Dataflow product only.
You can easily spot which job is the most expensive one ("event_raw_v1") and for which Dataflow SKUs we pay the most.

Very similar dashboard is prepared for the batch jobs deployed on ephemeral Dataproc clusters.

![Dataprod dashboard](/assets/images/finops_dataproc.webp)

There is a Dataproc specific SKU "Licencing fee" instead of "Streaming" and "Shuffle" in Dataflow.
The definition of the timeline is exactly the same, daily costs organized around jobs.

There is one more important advantage of the presented reports. 
GCP SKUs are very detailed (which is generally fine), for example every type of CPU is reported under different SKU.
But it introduces accidental complexity, on the presented dashboards everything is aggregated under single entry "CPU".
More convenient to use, when you want to focus on the costs not on the technical details.

### BigQuery, Pubsub and Cloud Storage reports

Dataproc or Dataflow are only a part of total data pipeline costs.
The next three dashboards are for the cost tracking of the most important "storages" used by the pipelines:

* BigQuery used by batch and streaming pipelines
* Pubsub used by streaming pipelines
* Cloud storage used mainly by batch pipelines as a staging area

On the BigQuery costs dashboard you can filter by the dataset name.
It is very important when the data pipeline stores results into more than one dataset.
The filtering is available only if the **allegro__dataset_name** label was set.

![BigQuery dashboard](/assets/images/finops_bigquery.webp)

On the Pubsub costs dashboard you can filter by the topic name.
It is very important when the data pipeline publishes to or subscribes on many topics.
The filtering is available only if the **allegro__topic_name** label was set.

![Pubsub dashboard](/assets/images/finops_pubsub.webp)

On the Cloud Storage dashboard you can filter by the bucket name.
It is very important when the data pipeline stores results into more than one bucket.
The filtering is available only if the **allegro__bucket_name** label was set.

![Cloud Storage dashboard](/assets/images/finops_gcs.webp)

## The challenges

Right now you should have a much better understanding of the difference between "resource" and "data pipelines" oriented billings.

TODO

## Scaling the discipline for the whole organization

TODO

## Summary

TODO
