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
Nowadays, taking care of costs efficiency is an integral part of every data engineer job.
I would like to share my own experiences with applying [FinOps](https://www.finops.org/introduction/what-is-finops/) discipline 
in the organization within tens of data engineering teams and thousands of data pipelines.

All my blog posts are based on the practical experiences, everything is battle tested and all presented examples are taken from the real systems.
And this time it will not be different.
On daily basis I manage the analytical data platform, the project with $100k+ monthly budget.
I will share how to keep costs of the streaming and batch pipelines on Google Cloud Platform under control.

## FinOps overview

FinOps is a very broad term, but what is especially important from a data engineer perspective?

* You take ownership of the cloud resources usage and costs.
  With great power of public cloud comes great responsibility.
* Cost is introduced as a regular metric of every data pipeline.
  More data and more processing power cause a higher costs.
* To optimize the costs you have to know detailed billing of every cloud resource.
  It does not make any sense to optimize the least expensive part of the system.
* Cost analysis becomes a part of [definition of done](https://www.agilealliance.org/glossary/definition-of-done/).
  After major changes you should also check the billings for any regression.
* Premature costs optimization is the root of all evil.
  Do you remember Donald Knuth paper about [performance](https://wiki.c2.com/?PrematureOptimization)?
  
## Google Cloud Platform resources

You may be surprised how complex is Google Cloud Platform billing.
There are dozens [products](https://cloud.google.com/products), every product has tens [SKUs](https://cloud.google.com/skus), hundreds of different elements to track and count.
What are the most important cost factors in the Google Cloud Platform for data pipelines I'm familiar with?

* Common resources
  - vCPU
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
  
There are other products relevant to the data pipelines, but I do not use them on daily basis:
* [Vertex AI](https://cloud.google.com/vertex-ai)
* [Bigtable](https://cloud.google.com/bigtable) (expensive beast)
* [Firestore](https://cloud.google.com/firestore)
* [Memory Store](https://cloud.google.com/memorystore)

## Resource oriented costs tracking

Because Google Cloud Platform billings are oriented around projects, products and SKUs, the built-in cost reports are focused on projects, products and SKUs as well.
Below you can find the real example of the report for one of my test environment, the costs for the last 30 days grouped by the cloud product.
As you can see "Compute Engine" is a dominant factor in the billing in this case.

![Billing dashboard](/assets/images/finops_billing_dashboard.webp)

Unfortunately, I must not enclose any financial details from the production environments because my employer is a [listed company](https://www.google.com/finance/quote/ALE:WSE).
If I had shown the real numbers I would have had severe troubles.
{: .notice--info}

The presented built-in [Cloud Billing Report](https://cloud.google.com/billing/docs/how-to/reports) page provides:

* Spending timeline within daily or monthly data granularity
* Change since the previous billing period 
* Ability to filter or split the total costs by projects, products or SKUs
* Insight into discounts, promotions and negotiated savings

At first, it looks that the cloud billing report plots everything you need, doesn't it?
Unluckily it has many shortcomings. 
Imagine the project with a dozen of different data pipelines using the same type of cloud resources.

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

1. Develop cloud resources [labelling convention](https://cloud.google.com/resource-manager/docs/creating-managing-labels#common-uses)
2. Apply the labels for ALL resources used by the data pipelines
3. Configure the [cloud billing data export](https://cloud.google.com/billing/docs/how-to/export-data-bigquery) to BigQuery
4. Craft Datastudio report aligned with the developed labelling convention, there is an [example](https://cloud.google.com/billing/docs/how-to/visualize-data)
5. Figure out the workarounds for GCP products which do not provide necessary details in the billing data export

I would say that 2) and 5) are the toughest parts of the journey.

### Labeling convention

There is no silver bullet, the best cloud resources labelling conventions does not exist.
You have to develop your own labeling convention, but I would like to share a few best practices:

* Introduce labelling naming convention at the very beginning
* Prepare and share the documentation, it should be clear for any adopter how to apply and how to interpret the labels
* Add a new labels only when they are really needed, too many labels do not help
* Continuously check the billing report for the resources without labels and fill the gaps
* Last but not least, use the prefix for all custom labels to easily recognize them from the built-in ones

Below you can find the labelling convention I have created in my company:

* **allegro__sc_id** - Every Allegro library, application or data pipeline is registered in the internal service catalog. 
  This is a central repository of metadata like business/technical owners, responsible team, issue tracker etc.
  The perfect identifier for cloud resources labelling.
* **allegro__job_name** - Every data pipeline could consist of many jobs, so the service identifier is not enough to identify every part of the pipeline.
  Job name is also more convenient to use than synthetic service identifier. 
  As you will see later, many dashboards are organized around the jobs names.
* **allegro__branch_name** - When you develop and test new feature it is handy to assign the branch/feature name as a label.
  You will exactly know the cost of the experimentation phase for this feature/branch.
  It is also quite useful during major upgrades, to verify that after change the job is still as cost-effective as before.
  Deploy the copy of the job aside the current production, and compare the costs using the label.

I also realized very quickly, that billing export does not provide any built-in labels for the well-known resources like BigQuery datasets, Pubsub topics or Cloud Storage buckets.
To mitigate this limitation the following additional labels are defined for the cloud resources used by the data pipelines:

* **allegro__topic_name** - The name of the Pubsub topic, apply the label to every topic of subscription
* **allegro__dataset_name** - The name of the BigQuery dataset, apply the label to every dataset
* **allegro__bucket_name** - The name of the Cloud Storage bucket, apply the label to every bucket

If the data pipelines subscribe or publish to multiple Pubsub topics, produce multiple BigQuery datasets or write to multiple Cloud Storage buckets 
the billing export will provide detailed costs for every single labelled resource.

### Summary reports

Before I go to the technical details I will present a few screens with the final reports I'm using on daily basis.
At the end, applying the labels for all resources in the cloud is a huge effort, so it must pay off somehow.

![Summary dashboard without shared costs](/assets/images/finops_summary2_dashboard.webp)

The summary dashboard for given project (sc-9936-nga-dev) with calculated costs for the last 30 days.
The total spending of $1.9k with the major participation from Dataproc and Dataflow.

The most interesting part of the dashboard is a colorful timeline with spending grouped by the jobs deployed in the project.
You will get the exact cost of every single data pipeline deployed in the project.
Easy to compare to each other, easy to spot any anomaly or trends.

Remember, due to the company policy I must not present reports from the production systems.
{: .notice--info}

The next step is to filter the report by job identifier, job name or feature branch.

![Summary dashboard for the single job](/assets/images/finops_summary3_dashboard.webp)

The job "event_v1" is implemented as Dataflow pipeline. 
It reads data from Pubsub, publishes results to Pubsub and also writes some data to BigQuery.
The costs structure is as follows:

* $61.8 for Dataflow, the largest share in the billing. 
  Streaming pipeline is deployed 24/7 so the cost is relatively high even for small virtual machines.
* $4.7 for reading and publishing from/to Pubsub
* $0.7 for writing to BigQuery
* $0.1 tiny amount for Cloud Storage, the costs of Dataflow staging bucket

This is the first report I check before applying any pipeline optimization.

### Dataflow report

To check more details about Dataflow data pipelines you need another report page.

![Dataflow dashboard](/assets/images/finops_dataflow_dashboard.webp)

On this page, the summary big numbers are focused on the most important Dataflow SKUs like: CPU, MEM, RAM, Local Disk, Streaming and Shuffle services.
Again there is also a timeline with costs grouped by Dataflow job name.

### Dataproc report

Similar report is also available for Spark jobs deployed on ephemeral Dataproc clusters.

![Dataprod dashboard](/assets/images/finops_dataproc_dashboard.webp)

The report presents the most important Dataproc SKUs and the timeline with costs grouped by Spark job name.

### BigQuery report

BigQuery detailed report is also similar to the Dataflow or Dataproc reports.

![BigQuery dashboard](/assets/images/finops_bigquery_dashboard.webp)

The main difference is that timeline is grouped by BigQuery dataset name instead of data pipeline job name.
To verify the cost of the given data pipeline just use the filter.
You will get the costs of BigQuery storage and queries for that pipeline split by dataset name (if the pipeline produces into more than one dataset).

### Pubsub report

Pubsub report is organized around topics.

![Pubsub dashboard](/assets/images/finops_pubsub_dashboard.webp)

You will get the costs split by topic, but you still are able to filter by data pipeline job (producer of data to the topic).

### Cloud Storage report

Google storage is organized around buckets.

![Pubsub dashboard](/assets/images/finops_storage_dashboard.webp)

You will get the costs split by bucket name, but you still are able to filter by data pipeline job (writer to the bucket).

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
