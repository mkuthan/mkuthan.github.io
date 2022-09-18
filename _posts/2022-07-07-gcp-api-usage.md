---
title: "Google Cloud Platform troubleshooting with API monitoring"
date: 2022-07-07
tags: [GCP, Performance, Dataproc, BigQuery, Terraform]
header:
    overlay_image: /assets/images/2022-07-07-gcp-api-usage/scott-webb-yekGLpc3vro-unsplash.webp
    caption: "[Unsplash](https://unsplash.com/@scottwebb)"
---

When you deploy any non-trivial data pipelines using complex infrastructure you should expect some troubles sooner or later.

* The pipeline could slow down
* Resources usage could be higher than usual
* For batch processing the throughput might be lower
* For real-time processing the overall latency might be higher

Today you will learn how to find the root cause of the performance problems in data pipelines deployed on Google Cloud Platform.

## The story

A few days ago on-duty engineer in my team reported some issues for [Spark](https://spark.apache.org/) jobs deployed on [Dataproc](https://cloud.google.com/dataproc) clusters. 
Large jobs got timeouts from the [Cloud Composer](https://cloud.google.com/composer), the smaller ones finished successfully, but it took 4-5 times longer than usual. 

After quick analysis we found that CPU utilization for Dataproc ephemeral cluster is fairly low:

![Dataproc low CPU utilization](/assets/images/2022-07-07-gcp-api-usage/dataproc-cpu-1.png)

For comparison, the same job on the same data before the incident:

![Dataproc normal CPU utilization](/assets/images/2022-07-07-gcp-api-usage/dataproc-cpu-2.png)

What could we do? Open a ticket to the Dataproc support that we observe degradation of managed service?

Wrong! If the root cause of the issue isn't in Dataproc itself, Dataproc support doesn't help.
{: .notice--warning}

When you fill the ticket for given Google Cloud Platform service (e.g. Dataproc), the support takes a look at internal metrics and logs of **this** service.
The support doesn't know anything about your Spark jobs deployed on the Dataproc cluster.
So if the root cause of the issue is in another part of Google Cloud Platform ecosystem, Dataproc support is ... useless.

## Troubleshooting

We know that our Spark jobs read data from [BigQuery](https://cloud.google.com/bigquery) using [Storage Read API](https://cloud.google.com/bigquery/docs/reference/storage), 
make some transformations and save the results to [Cloud Storage](https://cloud.google.com/storage).

If the job isn't able to read data at full speed, the CPU utilization will be low. 
It could be a Storage Read API issue or some network issue between Dataproc cluster and BigQuery.

Fortunately, almost every Google Cloud Platform API provides the following [metrics](https://cloud.google.com/apis/docs/monitoring):

* Request latency
* Requests count with response status details
* Request size
* Response size

Let's look at Storage Read API latency metrics:

![Storage Read API overall latency](/assets/images/2022-07-07-gcp-api-usage/storage-read-api-latency-1.png)

![Storage Read API by-method latency](/assets/images/2022-07-07-gcp-api-usage/storage-read-api-latency-2.png)

The strong evidence that the root cause of the Spark job slowness isn't in Dataproc but in BigQuery service!
{: .notice--info}

We filled a ticket to the BigQuery support, and quickly got confirmation that there is the global issue.
It was a problem with the latest change in "Global BigQuery Router" (whatever it is). 
The change had been rolled back after 2 days, and it solved the issue with our Spark jobs on the Dataproc cluster.

## Custom dashboards

The built-in dashboards for API metrics are quite good, but I would strongly recommend preparing dedicated dashboards for selected APIs only.
We defined reusable Terraform module for creating the custom dashboard for API metrics:

```terraform
resource "google_monitoring_dashboard" "dashboard" {
  dashboard_json = templatefile("${path.module}/templates/api.json",
    { title = var.title, service = var.service, method = var.method }
  )
}
```

Dashboard template:

```json
{
  "displayName": "Consumed API (${title})",
  "gridLayout": {
    "columns": "2",
    "widgets": [
      {
        "title": "Requests latencies",
        "xyChart": {
          "dataSets": [
            {
              "timeSeriesQuery": {
                "timeSeriesFilter": {
                  "filter": "metric.type=\"serviceruntime.googleapis.com/api/request_latencies\" resource.type=\"consumed_api\" resource.label.\"service\"=\"${service}\" resource.label.\"method\"=\"${method}\"",
                  "aggregation": {
                    "alignmentPeriod": "60s",
                    "perSeriesAligner": "ALIGN_SUM",
                    "crossSeriesReducer": "REDUCE_SUM"
                  }
                }
              },
              "plotType": "HEATMAP",
              "minAlignmentPeriod": "60s",
              "targetAxis": "Y1"
            }
          ],
          "timeshiftDuration": "0s",
          "yAxis": {
            "label": "y1Axis",
            "scale": "LINEAR"
          },
          "chartOptions": {
            "mode": "COLOR"
          }
        }
      },
      {
        "title": "Requests count",
        "xyChart": {
          "dataSets": [
            {
              "timeSeriesQuery": {
                "timeSeriesFilter": {
                  "filter": "metric.type=\"serviceruntime.googleapis.com/api/request_count\" resource.type=\"consumed_api\" resource.label.\"service\"=\"${service}\" resource.label.\"method\"=\"${method}\"",
                  "aggregation": {
                    "alignmentPeriod": "60s",
                    "perSeriesAligner": "ALIGN_MEAN",
                    "crossSeriesReducer": "REDUCE_MEAN",
                    "groupByFields": [
                      "metric.label.\"response_code\""
                    ]
                  }
                }
              },
              "plotType": "LINE",
              "minAlignmentPeriod": "60s",
              "targetAxis": "Y1"
            }
          ],
          "timeshiftDuration": "0s",
          "yAxis": {
            "label": "y1Axis",
            "scale": "LINEAR"
          },
          "chartOptions": {
            "mode": "COLOR"
          }
        }
      },
      {
        "title": "Request sizes",
        "xyChart": {
          "dataSets": [
           {
              "timeSeriesQuery": {
                "timeSeriesFilter": {
                  "filter": "metric.type=\"serviceruntime.googleapis.com/api/request_sizes\" resource.type=\"consumed_api\" resource.label.\"service\"=\"${service}\" resource.label.\"method\"=\"${method}\"",
                  "aggregation": {
                    "alignmentPeriod": "60s",
                    "perSeriesAligner": "ALIGN_PERCENTILE_95",
                    "crossSeriesReducer": "REDUCE_SUM"
                  }
                }
              },
              "plotType": "LINE",
              "minAlignmentPeriod": "60s",
              "targetAxis": "Y1"
            }
          ],
          "timeshiftDuration": "0s",
          "yAxis": {
            "label": "y1Axis",
            "scale": "LINEAR"
          },
          "chartOptions": {
            "mode": "COLOR"
          }
        }
      },
      {
        "title": "Response sizes",
        "xyChart": {
          "dataSets": [
            {
              "timeSeriesQuery": {
                "timeSeriesFilter": {
                  "filter": "metric.type=\"serviceruntime.googleapis.com/api/response_sizes\" resource.type=\"consumed_api\" resource.label.\"service\"=\"${service}\" resource.label.\"method\"=\"${method}\"",
                  "aggregation": {
                    "alignmentPeriod": "60s",
                    "perSeriesAligner": "ALIGN_PERCENTILE_95",
                    "crossSeriesReducer": "REDUCE_SUM"
                  }
                }
              },
              "plotType": "LINE",
              "minAlignmentPeriod": "60s",
              "targetAxis": "Y1"
            }
          ],
          "timeshiftDuration": "0s",
          "yAxis": {
            "label": "y1Axis",
            "scale": "LINEAR"
          },
          "chartOptions": {
            "mode": "COLOR"
          }
        }
      }
    ]
  }
}
```

Terraform module usage:

```terraform
module "dashboard_api_bq_storage_read" {
  source = "modules/monitoring-dashboard-api"

  title = "BQ Storage Read"
  service = "bigquerystorage.googleapis.com"
  method = "google.cloud.bigquery.storage.v1.BigQueryRead.ReadRows"
}

module "dashboard_api_bq_streaming_inserts" {
  source = "modules/monitoring-dashboard-api"

  title = "BQ Streaming Inserts"
  service = "bigquery.googleapis.com"
  method = "google.cloud.bigquery.v2.TableDataService.InsertAll"
}
```

Final dashboard for ReadRows method in Storage Read API. 
* Latency heatmap with percentiles (50th, 95th and 99th)
* Request count grouped by response status to see if there are errors
* Request and response sizes, it gives an information about batching

![Storage Read API custom dashboard](/assets/images/2022-07-07-gcp-api-usage/dashboard.png)

## Summary

Key takeaways:

* Distributed systems troubleshooting is hard, especially for managed service in the public cloud
* Ensure that you found the root cause of the problem before you open the support ticket
* Be prepared, monitor metrics of the most important Google Cloud APIs 
