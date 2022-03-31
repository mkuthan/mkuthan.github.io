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
  Develop, deploy, measure and then optimize if needed.
  Do you remember Donald Knuth's paper about [performance](https://wiki.c2.com/?PrematureOptimization)?
  
## Google Cloud Platform resources

You may be surprised how complex Google Cloud Platform billing is.
There are dozens of [products](https://cloud.google.com/products), every product has tens of [SKUs](https://cloud.google.com/skus), hundreds of different elements to track and count.
What are the most important cost factors in the Google Cloud Platform for data pipelines in my projects?

* Common resources
  - vCPU
  - Memory
  - Persistent disks
  - Local disks
  - Network egress 
* Dataflow
  - Streaming Engine (for real-time pipelines)
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
  
There are many other GCP products relevant to the data pipelines, so the cost structure could be different in your organization.
I would check the following products, not used by me on a daily basis.

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
* Does the cost of all resources utilized by the pipeline fit to its business case budget? 
* My billing report shows that most of the monthly budget is utilized by BigQuery, which data pipeline should be optimized first?
* Which resource causes the highest cost for the given data pipeline?

As a data engineer I'm interested in the total costs of the selected data pipeline.
Then I need to have an ability to drill down into the cost of every single resource used by this pipeline.

The data pipeline oriented cost reports are the missing part of the platform.
Resource oriented costs report are not sufficient.
{: .notice--info}

## Data pipeline oriented costs tracking

Now we have already defined the requirements for billing reports which helps us to get better costs understanding.
How to create data pipeline oriented costs tracking in Google Cloud Platform?

The recipe:

1. Develop cloud resources [labeling convention](https://cloud.google.com/resource-manager/docs/creating-managing-labels#common-uses)
2. Apply the labels for ALL resources used by the data pipelines
3. Configure the [cloud billing data export](https://cloud.google.com/billing/docs/how-to/export-data-bigquery) to BigQuery
4. Craft Datastudio report aligned with the developed labeling convention, there is public [example](https://cloud.google.com/billing/docs/how-to/visualize-data) available
5. Figure out the workarounds for GCP products which do not provide necessary details in the billing data export

I would say that 2) and 5) are the toughest parts of the journey.
{: .notice--info}

### Labeling convention

There is no silver bullet - perfect cloud resources labeling conventions do not exist.
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
  Deploy the copy of the job aside the current production, and compare the costs using this label.

I also realized very quickly that billing export does not provide any built-in labels for the well-known resources like BigQuery datasets, Pubsub topics or Cloud Storage buckets.
To mitigate this limitation the following additional labels are defined for the cloud resources used by the data pipelines:

* **allegro__topic_name** - The name of the Pubsub topic, apply the label to every topic and subscription
* **allegro__dataset_name** - The name of the BigQuery dataset, apply the label to every dataset
* **allegro__bucket_name** - The name of the Cloud Storage bucket, apply the label to every bucket

If the data pipelines subscribe or publish to multiple Pubsub topics, produce multiple BigQuery datasets or write to multiple Cloud Storage buckets 
the billing export will provide detailed costs for every labeled resource.

### Overview reports

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

Let start with the next report crafted for Dataflow real-time pipelines:

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

Right now you should have a much better understanding of the difference between "resources" and "data pipelines" oriented billings.
The final reports may look relatively simple but this is easier said than done.

* How to automate labeling to avoid gaps in the reports?
* What if the GCP product does not support labels at all?
* What if the labels set on the resources for unknown reasons are not available in the billing export?
* How to track costs of resources created automatically by managed GCP services? 
  Even if we set labels on the service itself they are not propagated to the underlying resources.
* What if you are using the libraries like Apache Beam or Apache Spark, and the API for setting labels is not publicly exposed?
* There is also a shared infrastructure like Cloud Composer or Cloud Logging, they also generate significant costs

Let's tackle all identified challenges one by one.

### Labeling automation

* Use [Terraform](https://www.terraform.io) for "static" cloud resources management like BigQuery datasets, Pubsub topics, Pubsub subscriptions and Cloud storage buckets.
Prepare the reusable modules with input parameters required to set the mandatory labels.
* Deliver GitHub [actions](https://docs.github.com/en/actions/learn-github-actions) or [composite actions](https://docs.github.com/en/actions/creating-actions/creating-a-composite-action) for deploying Dataflow streaming jobs.
The action will set all required labels every time when the job is deployed.
* Develop thin wrappers for [Dataproc Apache Operators](https://airflow.apache.org/docs/apache-airflow-providers-google/stable/operators/cloud/dataproc.html) or even better the [decorators](https://airflow.apache.org/docs/apache-airflow/stable/howto/create-custom-decorator.html).
They are responsible for setting all required labels when the batch job is deployed into Cloud Composer.

Looks complex and very time-consuming? It is for sure, but for dynamic resources like BigQuery queries the situation is even worse.
If you do not set any label on the [JobConfiguration](https://developers.google.com/resources/api-libraries/documentation/bigquery/v2/java/latest/com/google/api/services/bigquery/model/JobConfiguration.html#setLabels-java.util.Map-)
the costs of all queries are just aggregated under "BigQuery / Analysis" SKU. 

### BigQuery jobs

Fortunately all BigQuery jobs are reported in the BigQuery [information schema](https://cloud.google.com/bigquery/docs/information-schema-jobs).
In the information schema you can find the query expression and "total_bytes_billed" column with billed bytes.
As long as cost per TiB is well [known](https://cloud.google.com/bigquery/pricing#on_demand_pricing), the final cost of the query may be estimated. 
Instead of labeling every single query it is easier to prepare the estimated costs report based on queries found in the information schema.

![BigQuery jobs dashboard](/assets/images/finops_bigquery_jobs.webp)

Although, there are at least two disadvantages:

* No direct connection with the pipelines, you have to manually "assign" the query to the pipeline. 
  Not a big deal for the data engineer who is the author of the queries.
* Daily or hourly jobs do not generate exactly the same query on every run. 
  The time related expressions are typically varying and need to be normalized

You can use the following regular expression for query normalization:

```
REGEXP_REPLACE(query, '\\d{8,10}|\\d{4}-\\d{2}-\\d{2}([T\\s]\\d{2}:\\d{2}:\\d{2}(\\.\\d{3})?)?', 'DATE_PLACEHOLDER')
```

### BigQuery Storage API

There is at least one crucial GCP product without support for labels: [BigQuery Storage API](https://cloud.google.com/bigquery/docs/reference/storage/libraries).
All costs are aggregated under "BigQuery Storage API / [Write | Read]" SKUs, 
very similar situation to the BigQuery Jobs when all costs go to the "Analysis" SKU.
There is an [open issue](https://issuetracker.google.com/185163366) in the bug tracker.

Unfortunately, BigQuery Storage API does not expose any details in the BigQuery information schema to estimate the costs like for BigQuery jobs.
I do not understand how the BigQuery Storage API has got GA status if such limitations still exist.

I have not found any workaround to estimate the costs for BigQuery Storage API, yet. 
Be sure to let me know if you know any.
{: .notice--info}

### BigQuery 3rd party APIs

The most data pipelines are not implemented within the BigQuery API directly but with some 3rd party, higher level API.
The high level APIs need to expose the underlying BigQuery API for setting the labels.

* Apache Beam still does not support job labels on BigQuery read/write transforms [BEAM-9967](https://issues.apache.org/jira/browse/BEAM-9967)
* BigQuery Spark Connector has recently got the support for setting job labels [PR-586](https://github.com/GoogleCloudDataproc/spark-bigquery-connector/pull/568)
* Spotify Scio has support for the labels for the long time, but only if the Scio specific BigQuery connector is used [PR-3375](https://github.com/spotify/scio/pull/3375)

### BigQuery tables

Do you know why BigQuery report for storage costs is organized around datasets, not tables?
Because the labels on the BigQuery tables are not available in the billing export, only labels from datasets are exported.
Due to this limitation, I have many fine-grained datasets in the project, just to get proper accountability.

Please vote for the following [issue](https://issuetracker.google.com/issues/227218385).

### Dataflow internal subscriptions

When you make a Pubsub subscription for a Dataflow job 
and configure [timestamp attribute](https://beam.apache.org/releases/javadoc/2.37.0/org/apache/beam/sdk/io/gcp/pubsub/PubsubIO.Read.html#withTimestampAttribute-java.lang.String-) for watermark tracking
additional tracking subscription is created.
See the [official documentation](https://cloud.google.com/dataflow/docs/concepts/streaming-with-cloud-pubsub#high_watermark_accuracy) 
or [stack overflow question](https://stackoverflow.com/questions/42169004/what-is-the-watermark-heuristic-for-pubsubio-running-on-gcd) for more details.
Unfortunately the tracking subscriptions do not get labels from Dataflow jobs nor original subscriptions.

Please vote for the following [issue](https://issuetracker.google.com/issues/227218387).

Workaround: the internal subscriptions cost exactly like the original subscriptions, so you can easily estimate the total costs.
{: .notice--info}

### Shared infrastructure

The costs of shared infrastructure can not be assigned to any particular data pipeline. 
Just stay with the "resource oriented" billing and try to minimize the overall costs.
From my experience there are two of the most expensive shared infrastructure costs on Google Cloud Platform for data pipelines.

* [Cloud Composer](https://cloud.google.com/composer) -- if you want to optimize Cloud Composer I highly recommend reading one of my blog posts - [GCP Cloud Composer 1.x tuning](/blog/2022/03/15/gcp-cloud-composer-tuning/)
* [Cloud Logging](https://cloud.google.com/logging) -- to minimize the costs, minimize the number of log entries produced by the data pipelines and their infrastructure

Below you find the number of log entries from my production environment for the one day.

![Number of log entries](/assets/images/finops_logging.webp)

As you can see, a lot of logs come from the Dataproc ephemeral clusters. 
The best thing you can do is to apply [exclusion filters](https://cloud.google.com/logging/docs/routing/overview#exclusions) on the log router. 
Filtered logs are not counted in the billings.

If you have any other practices for shared infrastructure cost optimizations, let me know in the comments.
{: .notice--info}

## Summary

I fully realize that FinOps discipline is not easy to adopt for data engineers.
Finally, we would like to develop data pipelines, not to think about billings, budgets and overspending.

But in the public cloud era there is no choice, you have to monitor costs of the data pipelines in exactly the same way as you monitor overall health, latency or throughput.
The built-in cloud billing tools organized around products do not help a lot.
To get detailed costs reports you have put a lot of effort to create data pipeline oriented costs monitoring.

I hope that this blog post gives you some ideas on how to develop your own toolset for applying FinOps discipline.
The sooner the discipline is adopted the more savings could be expected.