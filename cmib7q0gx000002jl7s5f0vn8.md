---
title: "Keep an Eye on the Costs"
datePublished: Sun Nov 23 2025 04:23:11 GMT+0000 (Coordinated Universal Time)
cuid: cmib7q0gx000002jl7s5f0vn8
slug: keep-an-eye-on-the-costs
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1763845280972/c4917545-d401-41d4-a110-7f6198e55898.png
tags: aws, cost, splunk, cost-optimisation, costsavings

---

Modern software architecture decisions often emphasize scalability, reliability, and performance, but cost is frequently overlooked until the bill arrives. For distributed systems, especially those involving cloud services, cost should be treated as a first-class technical requirement.

In this article, we analyze a real-world problem from a cost-first perspective and walk through three alternative implementations step by step. The goal is to help software developers internalize a repeatable method for cost analysis and provide practical techniques to prevent unexpected cloud bills.

## The Problem

We have a public Single-Page Application (SPA) that stores logs locally in [IndexedDB](https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API). A [Service Worker](https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API) collects and sends logs every 5 minutes, with approximately:

* **Request size**: ~20 log entries per request.
    
* **Request weight**: ~40 KB per request.
    
* **Scale**: 2,000 instances running 12 hours daily.
    

This results in:

* **Requests per instance per day**: 12 hours × (60 / 5) = 144 requests.
    
* **Total requests per day**: 144 × 2000 = 288,000 requests.
    
* **Total requests per month**: 288,000 × 30 = 8,640,000 requests.
    
* **Daily log volume**: 288,000 × 40 KB = 11.52 GB.
    

Logs must ultimately reach [Splunk](https://www.splunk.com/), which is only accessible privately. We are evaluating three architectural destinations to receive logs from the SPA.

## Option 1: AWS Lambda (256MB)

An AWS Lambda function receives the request, processes it, and forwards it to Splunk over the private network.

**Advantages:**

* Zero infrastructure management.
    
* Auto-scaling without configuration.
    
* Built-in monitoring and logging.
    

**Disadvantages:**

* API Gateway adds latency.
    
* Cold starts can affect P99 latency.
    

### Cost Analysis

* **AWS Lambda Invocations:**
    
    * [**Cost**](https://aws.amazon.com/lambda/pricing/): $0.20 per 1 million requests.
        
    * **Total Invocation Cost**: 8,640,000 requests \* $0.20/1,000,000 requests = $1.73
        
* **AWS Lambda Compute Time:**
    
    * **Average Duration**: 150 ms per request.
        
    * **Total Compute (seconds)**: 8,640,000 requests \* 150 ms/request = 1,296,000,000 ms = 1,296,000 seconds.
        
    * **Total Compute (GB-seconds)**: 1,296,000 seconds \* (256/1024) GB = 324,000 GB-seconds.
        
    * [**Cost**](https://aws.amazon.com/lambda/pricing/): $0.0000166667 per GB-second.
        
    * **Total Compute Cost**: 324,000 GB-seconds \* $0.0000166667/GB-second = $5.40
        
* **AWS API Gateway (HTTP API)**
    
    * [Cost](https://aws.amazon.com/api-gateway/pricing/?nc1=h_ls): $1.00 per million requests.
        
    * **Total Requests Cost**: 8,640,000 requests × $1/1,000,000 requests = $8.64
        
* **Total Cost**: $1.73 + $5.40 + $8.64 = $15.77
    

### Option 2: Kubernetes Pod (256Mi, 256m CPU)

A lightweight API deployed as a Pod in an existing Kubernetes cluster.

**Advantages:**

* Lower per-request cost when using an existing cluster.
    

**Disadvantages:**

* Requires cluster management expertise.
    
* More complex deployment pipeline.
    

### Cost Analysis

> **EKS Control Plane** and ALB costs will not be included.

* **Pod resource allocation**:
    
    * **Node Type**: [`c7i-flex.xlarge`](https://instances.vantage.sh/aws/ec2/c7i-flex.xlarge?currency=USD) (4 vCPU, 8 GB RAM).
        
    * [**Cost**](https://instances.vantage.sh/aws/ec2/c7i-flex.xlarge?currency=USD&duration=hourly)**(On-Demand)**: $0.17/hour = $122.4/month.
        
    * **CPU Pod Utilization**: 0.256 vCPU / 4 vCPU = 6.4%
        
    * **Memory Pod Utilization**: 256 MB / 8192 MB = 3.1%
        
    * **Allocated Cost by CPU**: $122.4 × 0.064 = $7.83
        
    * **Allocated Cost by Memory**: $122.4 × 0.031 = $3.79
        
    * **Total Allocated Cost**: max($7.83, $3.79) = $7.83
        
* **Total Cost**: $7.83
    

### Option 3: Upload Logs to S3

The service worker uploads directly to S3 using pre-signed URLs. A message is sent to an SQS queue, which Splunk listens to. Splunk reads from S3, and the logs expire after one day.

**Advantages:**

* No compute resources needed.
    

**Disadvantages:**

* Splunk must be configured to read from S3.
    
* Less real-time than push models.
    

### Cost Analysis

* **Upload Costs (PUT requests)**:
    
    * [**Cost**](https://aws.amazon.com/s3/pricing/)**:** $0.005 per 1,000 requests.
        
    * **Total Upload Cost:** 8,640,000 requests \* $0.005/1,000 requests = $43.20
        
* **Storage Costs:**
    
    * Since files are deleted after one day, the average daily storage is just the amount uploaded daily.
        
    * [**Cost**](https://aws.amazon.com/s3/pricing/)**(S3 Standard Storage):** $0.023 per GB.
        
    * **Total Storage Cost:** 11.52 GB \* $0.023/GB = $0.26
        
* **Amazon SQS Costs:**
    
    * **Messages Sent from S3 to SQS:** 8,640,000 `SendMessage` requests.
        
    * **Polling by Splunk (**every 10 seconds**):** 8,640 polls/day \* 30 days = 2,592,000 `ReceiveMessage` requests.
        
    * **Messages Deleted by Splunk:** 8,640,000 `DeleteMessage` requests.
        
    * [**Cost**](https://aws.amazon.com/sqs/pricing/)**:** $0.40 per 1 million requests.
        
    * **Total SQS Cost:** 19,872,000 requests \* $0.40/1,000,000 requests = $7.94
        
* **Download Costs (GET requests):**
    
    * [**Cost**](https://aws.amazon.com/s3/pricing/)**: $**0.0004 per 1,000 requests.
        
    * Total Download Cost: 8,640,000 requests \* $0.0004/1000 requests = **$**3.45
        
* **Total Cost**: $43.20 + $0.26 +$7.94 + **$**3.45 = $54.85
    

## Conclusions

**The Kubernetes Pod is the lowest-cost solution** at **$7.83/month**, assuming the organization already maintains an EKS cluster. The operational cost is low only because cluster and ALB expenses are excluded; otherwise, the economics change significantly. This option is best suited for teams with Kubernetes expertise and existing cluster capacity.

**AWS Lambda offers the best balance between low cost and zero operational overhead.**  
At **$15.77/month**, it is inexpensive, scales transparently, and requires no infrastructure management. For most teams, this is the most practical and maintainable trade-off.

**S3 ingestion is the most expensive (and perhaps the most elegant) option** at **$54.85/month**, primarily because PUT requests scale linearly with the number of batches. While it eliminates compute concerns, it adds complexity (S3 → SQS → Splunk) and is less real-time. This model only makes sense if Splunk’s S3 ingestion pipeline is strategically preferred or if we increase the batch frequency to at least 60 minutes.

If minimizing cloud spend is the absolute priority and Kubernetes capacity already exists, the second option provides the lowest cost. Otherwise, Lambda gives the best cost-to-operational-simplicity ratio.

Thanks, and happy coding.