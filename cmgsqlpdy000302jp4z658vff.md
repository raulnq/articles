---
title: "Querying AWS Application Load Balancer Logs"
datePublished: Thu Oct 16 2025 01:24:23 GMT+0000 (Coordinated Universal Time)
cuid: cmgsqlpdy000302jp4z658vff
slug: querying-aws-application-load-balancer-logs
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1760568449086/982fc49f-64d8-4f0a-b7db-31437318d182.png
tags: aws, aws-glue, aws-athena

---

AWS Application Load Balancer (ALB) logs are a critical source of information for troubleshooting, security analysis, and performance optimization. These logs capture detailed information about every request processed by our load balancer, including client IP addresses, request paths, response codes, latencies, and more.

However, raw ALB logs stored in S3 are difficult to analyze at scale. They are stored as compressed text files, organized by date and time, making manual inspection impractical for high-traffic applications. This is where querying capabilities become essential.

## Understanding the AWS Components

Before we start the implementation, let's quickly go over the AWS components we will be discussing:

### AWS Athena

[AWS Athena](https://docs.aws.amazon.com/athena/latest/ug/what-is.html) is a serverless, interactive query service that allows us to analyze data directly in Amazon S3 using standard SQL. We don't need to set up or manage any infrastructure—we simply define our table schema and start querying.

For AWS ALB logs, Athena excels because:

* It scales automatically with our query complexity
    
* We only pay for the data scanned (typically $5 per TB)
    
* It integrates natively with S3, where AWS ALB logs are stored
    

### AWS Glue Data Catalog

The [AWS Glue Data Catalog](https://docs.aws.amazon.com/prescriptive-guidance/latest/serverless-etl-aws-glue/aws-glue-data-catalog.html) is a centralized metadata repository that stores table definitions, schemas, and partition information. It acts as a persistent metadata store for our data lakes. When we create a table in AWS Athena, we are actually creating an entry in the AWS Glue Data Catalog.

Think of it as a database catalog that tells AWS Athena where our data lives in S3, what format it's in, and how to interpret it—without moving or copying the actual data.

### AWS Glue Crawler

[AWS Glue Crawler](https://docs.aws.amazon.com/glue/latest/dg/add-crawler.html) is an automated service that scans our data sources (like S3 buckets), infers schemas, and automatically creates or updates table definitions in the AWS Glue Data Catalog. It's useful for discovering data structures, but has limitations we'll discuss next.

### Partitions in AWS Glue / Athena

In AWS Glue and Athena, which both use the same Data Catalog, a [partition](https://docs.aws.amazon.com/athena/latest/ug/partitions.html) is a logical subdivision of a dataset, often based on a key like date. Without partitions, AWS Athena scans every file in the S3 path for each query, leading to higher costs and lower performance.

## Solution Approaches

There are three main approaches to querying AWS ALB logs with AWS Athena:

### AWS Glue Crawler (Automated Schema Discovery)

**How it works**: AWS Glue Crawler scans our S3 bucket containing AWS ALB logs, automatically detects the schema, and creates a table in the AWS Glue Data Catalog.

**Limitation**: While the AWS Glue Crawler can discover partitions during its run, it automatically infers partitions only if the folder names follow the `key=value` format, such as:

```bash
s3://<bucket>/.../year=2025/month=10/day=15/
```

That is the [Hive-style](https://athena.guide/articles/hive-style-partitioning) partition naming convention that AWS Glue understands natively. Unfortunately, AWS ALB log paths look like this:

```bash
s3://<bucket>/<prefix>/AWSLogs/<account-id>/elasticloadbalancing/<region>/2025/10/15/
```

### AWS Lambda with S3 Event Triggers (Automated Partitioning)

**How it works**: This approach combines AWS Glue Crawler with AWS Lambda functions. When new AWS ALB log files arrive in S3, an event trigger invokes a function that creates the corresponding partition in the catalog.

**Limitation**: While this solves the partition issue, it introduces unnecessary complexity.

* Executions are triggered for every log file AWS ALB writes (which can be hundreds or thousands per hour in high-traffic scenarios).
    
* Most of these executions are redundant since partitions are typically organized by day or hour.
    

### AWS Athena with Partition Projection (Recommended)

**How it works**: Partition projection is an AWS Athena feature that dynamically generates partition information at query time without storing it in the AWS Glue Data Catalog. We define partition patterns in the table properties, and AWS Athena calculates which partitions to read based on our query filters. So, this is the solution we will explain in detail.

Assuming we already have our AWS ALB writing logs to S3. Create an AWS Athena table that uses partition projection. Execute this SQL in the Athena query editor:

```sql
CREATE EXTERNAL TABLE IF NOT EXISTS alb_logs (
    type string,
    time string,
    elb string,
    client_ip string,
    client_port int,
    target_ip string,
    target_port int,
    request_processing_time double,
    target_processing_time double,
    response_processing_time double,
    elb_status_code int,
    target_status_code string,
    received_bytes bigint,
    sent_bytes bigint,
    request_verb string,
    request_url string,
    request_proto string,
    user_agent string,
    ssl_cipher string,
    ssl_protocol string,
    target_group_arn string,
    trace_id string,
    domain_name string,
    chosen_cert_arn string,
    matched_rule_priority string,
    request_creation_time string,
    actions_executed string,
    redirect_url string,
    error_reason string,
    target_port_list string,
    target_status_code_list string,
    classification string,
    classification_reason string
)
PARTITIONED BY (
    year string,
    month string,
    day string
)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.RegexSerDe'
WITH SERDEPROPERTIES (
    'serialization.format' = '1',
    'input.regex' = '([^ ]*) ([^ ]*) ([^ ]*) ([^ ]*):([0-9]*) ([^ ]*)[:-]([0-9]*) ([-.0-9]*) ([-.0-9]*) ([-.0-9]*) (|[-0-9]*) (-|[-0-9]*) ([-0-9]*) ([-0-9]*) \"([^ ]*) ([^ ]*) (- |[^ ]*)\" \"([^\"]*)\" ([A-Z0-9-]+) ([A-Za-z0-9.-]*) ([^ ]*) \"([^\"]*)\" \"([^\"]*)\" \"([^\"]*)\" ([-.0-9]*) ([^ ]*) \"([^\"]*)\" \"([^\"]*)\" \"([^ ]*)\" \"([^\s]+?)\" \"([^\s]+)\" \"([^ ]*)\" \"([^ ]*)\"'
)
LOCATION 's3://<bucket>/<prefix>/AWSLogs/<account-id>/elasticloadbalancing/<region>/'
TBLPROPERTIES (
    'projection.enabled' = 'true',
    'projection.year.type' = 'integer',
    'projection.year.range' = '2025,2099',
    'projection.year.digits' = '4',
    'projection.month.type' = 'integer',
    'projection.month.range' = '01,12',
    'projection.month.digits' = '2',
    'projection.day.type' = 'integer',
    'projection.day.range' = '01,31',
    'projection.day.digits' = '2',
    'storage.location.template' = 's3://<bucket>/<prefix>/AWSLogs/<account-id>/elasticloadbalancing/<region>/${year}/${month}/${day}'
);
```

**Schema Definition**: The column definitions match the AWS ALB log format. Each field corresponds to a specific piece of information in the log entry (client IP, response codes, timing, etc.). AWS ALB logs follow a specific format documented by AWS.

**PARTITIONED BY**: We define three partition columns: `year`, `month`, and `day`. These align with how AWS ALB organizes logs in S3. Partitioning is crucial because it allows Athena to skip scanning irrelevant data, dramatically reducing query costs and improving performance.

**ROW FORMAT SERDE**: We use `RegexSerDe` to parse the log entries. AWS ALB logs are space-delimited with quoted strings, and this regex pattern extracts each field correctly. The regex handles edge cases like missing values (represented by `-`) and quoted fields that may contain spaces.

**LOCATION**: This points to the base path where AWS ALB writes logs. Note that it doesn't include the year/month/day folders—those are handled by partition projection.

**TBLPROPERTIES**: Partition projection configuration.

* `projection.enabled = 'true'`: Activates partition projection for this table.
    
* `projection.year.type = 'integer'`: Defines year as an integer partition. The range `2025,2099` covers the expected lifespan of your logs. Adjust these values based on your needs.
    
* `projection.month.type = 'integer'`: Defines month as an integer partition. The range `1,12` covers all months. The `digits = '2'` ensures single-digit months are zero-padded (01, 02, etc.) to match S3 paths.
    
* `projection.day.type = 'integer'`: Similar to month, covers days 1-31 with zero-padding.
    
* `storage.location.template`: This is critical; it tells AWS Athena how to construct S3 paths for each partition combination. The variables `${year}/${month}/${day}` are replaced with actual values at query time.
    

Now you can query our logs using standard SQL. Here are practical examples:

**Find all 500 errors in the last 24 hours.**

```sql
SELECT 
    time,
    client_ip,
    request_verb,
    request_url,
    elb_status_code,
    target_status_code,
    target_ip
FROM alb_logs
WHERE year = '2025'
  AND month = '10'
  AND day = '15'
  AND elb_status_code = 500
ORDER BY time DESC;
```

**Explanation**: This query filters by partition columns first (`year`, `month`, `day`), which is essential for performance. Athena will only scan files from October 15, 2025, ignoring all other days. Always filter by partition columns when possible to minimize data scanned.

**Analyze request latency by endpoint.**

```sql
SELECT 
    request_url,
    COUNT(*) as request_count,
    ROUND(AVG(target_processing_time), 3) as avg_target_time,
    ROUND(AVG(request_processing_time), 3) as avg_request_time,
    ROUND(AVG(response_processing_time), 3) as avg_response_time,
    ROUND(MAX(target_processing_time), 3) as max_target_time
FROM alb_logs
WHERE year = '2025'
  AND month = '10'
  AND day >= '14'
  AND day <= '15'
GROUP BY request_url
HAVING COUNT(*) > 100
ORDER BY avg_target_time DESC;
```

**Explanation**: This query analyzes performance across multiple days (14th and 15th). It calculates average and max latencies for each endpoint. The `HAVING COUNT(*) > 100` ensures we only look at endpoints with significant traffic, avoiding skewed statistics from rarely-hit URLs.

**Identify top client IPs by request volume**

```sql
SELECT 
    client_ip,
    COUNT(*) as request_count,
    COUNT(DISTINCT request_url) as unique_urls,
    SUM(received_bytes) as total_received_bytes,
    SUM(sent_bytes) as total_sent_bytes
FROM alb_logs
WHERE year = '2025'
  AND month = '10'
  AND day = '15'
GROUP BY client_ip
ORDER BY request_count DESC;
```

**Explanation**: Useful for identifying potential DDoS attacks or heavy users. This query groups by client IP and aggregates request counts and bandwidth usage. The `unique_urls` metric can help distinguish between legitimate users and suspicious scanning behavior (many requests to diverse URLs often indicate reconnaissance).

Thanks, and happy coding.