---
title: "Estimating Observability Costs in AWS"
datePublished: Thu Aug 15 2024 00:03:24 GMT+0000 (Coordinated Universal Time)
cuid: clzuiotks00000albbebo6zvk
slug: estimating-observability-costs-in-aws
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1723680004064/730cfca0-ef33-4cae-b492-8605bec7917e.png
tags: aws, costs, observability, aws-cloudwatch, aws-x-ray

---

Observability is a crucial practice that allows us to understand a system's behavior from the outside by leveraging logs, traces, and metrics. Over the years, it has become easier to implement due to the maturity of cloud providers and industry standards. However, this ease of integration can make us overlook the associated costs, which often go unnoticed until the monthly bill arrives.

In this post, we will walk through the process of estimating basic observability costs using AWS CloudWatch and AWS X-Ray for an API that will handle 1,500,000 requests per day and has the following non-functional requirements:

* Log the request and response body (the log entry in UTF-8 format has an average size of 0.25 KB).
    
* Each log entry should be available for three months.
    
* Tracing is needed for 10% of the requests.
    
* Five metrics will be published every minute.
    
* An alarm will be set up for each metric.
    
* A single dashboard will be used to visualize all the metrics.
    

> Free tier quotas are not considered during the calculations.

## Logs

For [AWS CloudWatch Logs](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/WhatIsCloudWatchLogs.html), the main factors contributing to the cost are ingestion, storage, retrieval, and transfer. The retrieval and transfer costs depend on how the logs are used, so we will ignore them in our calculation.

* Average size of a log event = 0.25 KB
    
* Number of log events per day = 1,500,000
    
* Total data volumen per month = 30 days \* 1,500,000 events \* 0.25 KB = 10.7 GB
    
* Ingestion costs = 10.7 GB \* **$0.50 per GB** = $5.35
    
* Log retention period = 3 months
    
* Log compression factor = 0.15
    
* Storage cost = 10.7 GB \* 0.15 \* 3 months \* **$0.03 per GB** = $0.14445
    

The number of log events per day is derived from the number of requests. Therefore\*\*\*,\*\*\* the total monthly cost to ingest and store the logs for three months is $5.49445.

## Tracing

For tracing, the main cost factors when using [AWS X-Ray](https://docs.aws.amazon.com/xray/latest/devguide/aws-xray.html) are recording, scanning, and retrieval. Since scanning and retrieval costs depend on trace usage, we will ignore them in our calculation.

* Number of traces per day = 1,500,000
    
* Sampling rate = 0.10
    
* Total traces per month = 1,500,000 \* 0.10 \* 30 days = 4,500,000
    
* Recording costs = 4,500,000 \* **$0.000005** = $22.50
    

The number of traces per day is equal to the number of requests. Therefore, the total monthly cost to record traces is $22.50.

## Metrics

The cost structure of [AWS CloudWatch](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/working_with_metrics.html) depends on the number of metrics per month and the calls to the `PutMetricData` API needed to push the data points. Alarms and dashboard costs, like metrics, depend solely on the number of instances per month.

> AWS CloudWatch treats each unique combination of dimensions as a separate metric, even if the metrics have the same name. For example, a metric with one dimension and a cardinality of four is considered four metrics from a cost perspective. If we add another dimension with a cardinality of three, we now have twelve metrics.

* Number of metrics per month = 5
    
* Metrics costs = 5 \* $0.30 = $1.50
    
* API call frequency = 1 per minute
    
* API calls per month = 5 \* 1 \* 60 minutes \* 24 hours \* 30 days = 216,000
    
* API costs = 216,000 \* **$0.00001** = $2.16
    
* Number of alarms per month = 5
    
* Alarm costs = 5 \* **$0.10** = $0.50
    
* Number of dashboards per month = 1
    
* Dashboard costs = 1 \* **$3.00** = $3.00
    

Therefore, the monthly cost for metrics, alarms, and dashboards is $7.16. Using Embedded Metric Format (EMF) to embed metrics directly within structured log events replaces the API costs with the log costs mentioned earlier.

## Conclusions

By breaking down the costs associated with logs, tracing, and metrics, we can better understand where our budget is going. This helps us optimize our cloud expenses and avoid unexpected costs. Please check the updated [AWS CloudWatch](https://aws.amazon.com/cloudwatch/pricing/?nc1=h_ls) prices and the [AWS X-Ray](https://aws.amazon.com/xray/pricing/) prices before doing your own calculations. Thanks, and happy coding.