---
title: "AWS Lambda Function: Check List"
datePublished: 2026-04-03T22:03:46.894Z
cuid: cmnjgajig00432dk499sq6xdv
slug: aws-lambda-function-check-list
cover: https://cdn.hashnode.com/uploads/covers/625b81432f84e0d4fc33dca5/388d992e-1f84-4c4d-990c-0dbbe4cbe650.png
tags: lambda, aws, dotnet

---

AWS Lambda has become a cornerstone of modern serverless architectures, but writing a function that "just works" is very different from writing one that is performant, cost-effective, secure, and ready for production at scale. For .NET developers, the distance between a naive implementation and a well-engineered one is surprisingly wide — it spans everything from cold start times and memory allocation, to secret management, deployment safety, and event loss prevention.

This guide covers ten areas of Lambda excellence for .NET, each derived from real-world production patterns and common failure modes. It assumes you are comfortable with the basics of Lambda and the AWS .NET SDK, and focuses on the *why* behind each practice as much as the *how*.

* * *

## Runtime & Deployment Model

### Use .NET 10 (or Latest LTS) on arm64

**What it is:** AWS Graviton2 processors power arm64 Lambda functions. They offer better performance-per-dollar for most workloads, and AWS charges approximately 20% less per GB-second compared to x86.

**Why it matters:** For a high-traffic function with 500ms average duration at 512 MB memory, switching from x86 to arm64 can save over $15/month per million daily invocations with zero code changes.

```yaml
# template.yaml
Globals:
  Function:
    Runtime: dotnet10
    Architectures:
      - arm64
    MemorySize: 512

Resources:
  MyFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: MyFunction::MyFunction.Function::FunctionHandler
      CodeUri: src/MyFunction/
```

* * *

### Minimize Deployment Package Size

**What it is:** The deployment package is the ZIP artifact containing your compiled application and its dependencies, uploaded to Lambda via S3. Its size directly affects how long Lambda takes to extract and load your code during a cold start.

**Why it matters:** Lambda extracts and initializes your deployment package on every cold start. A 50 MB package takes significantly longer to initialize than a 5 MB one. Smaller packages also reduce S3 storage costs and deployment time.

* * *

### Use Native AOT Compilation

**What it is:** Ahead-of-Time (AOT) compilation converts your .NET code directly into a native binary at build time, eliminating the Just-In-Time (JIT) compiler that normally runs when a .NET application starts. For Lambda, this means the initialization phase — the most expensive part of a cold start — is dramatically faster.

**Why it matters:** Lambda bills in 1ms increments. Cold starts are fully billed duration. On a JIT-based .NET function, initialization can take 400–1200ms depending on the number of loaded assemblies and SDK clients. With Native AOT, this drops to 30–80ms.

* * *

### Use Lambda SnapStart (if not using AOT)

**What it is:** SnapStart takes a snapshot of the initialized execution environment after the `Init` phase completes and caches it. Subsequent cold starts restore from the snapshot instead of re-running initialization, reducing cold start latency by up to 90%.

**Why it matters:** If you have an existing JIT-based .NET function that you cannot easily migrate to AOT (e.g., it uses libraries incompatible with trimming), SnapStart delivers most of the cold start benefit without a code rewrite.

* * *

## Memory & CPU Sizing

### Profile with AWS Lambda Power Tuning

**What it is:** The [AWS Lambda Power Tuning](https://github.com/alexcasalboni/aws-lambda-power-tuning) tool is an open-source Step Functions state machine that runs your function at multiple memory configurations (e.g., 128 MB, 256 MB, 512 MB, 1024 MB, 1769 MB, 3008 MB) and returns a cost and performance graph showing the optimal setting.

**Why it matters:** The relationship between memory and cost is not linear because CPU allocation scales with memory. A function that runs in 1000ms at 256 MB might run in 200ms at 1024 MB, making the higher-memory option cheaper despite a higher per-ms rate.

* * *

### Don't Under-Allocate Memory

**What it is:** Lambda allocates CPU proportionally to memory. Setting memory too low starves your function of CPU, which can make it run slower and cost more despite the lower per-GB-second rate.

| Memory | CPU Allocation |
| --- | --- |
| 128 MB | ~5% of a vCPU |
| 512 MB | ~25% of a vCPU |
| 1769 MB | 1 full vCPU |
| 3584 MB | 2 full vCPUs |

**Why it matters:** A .NET function with heavy computation (JSON parsing, data transformation, LINQ operations) starved of CPU at 256 MB may run 4–8x slower than at 1769 MB, making it cost more despite the lower RAM pricing.

* * *

### Set Memory Based on Actual Measurements

**What it is:** Every Lambda invocation emits a `REPORT` log line containing the actual peak memory used. This measured value is the correct baseline for sizing your memory allocation, rather than guessing.

**Why it matters:** Setting memory allocation to peak measured usage plus 15% headroom prevents out-of-memory errors while avoiding wasteful over-provisioning. Both extremes cost money: OOM errors cause failed invocations and retries; excessive headroom means you pay for memory you never use.

```plaintext
REPORT RequestId: abc123  Duration: 245.12 ms  Billed Duration: 246 ms
Memory Size: 512 MB  Max Memory Used: 178 MB
```

Set your memory allocation to `peakMB * 1.15` (15% headroom above peak measured usage).

* * *

## Cold Start Optimization

### Move Heavy Initialization Outside the Handler

**What it is:** In Lambda, code that runs at class construction or in static initializers executes only once per execution environment lifecycle — during the `Init` phase — not on every invocation. By moving expensive setup (SDK client construction, config loading, connection establishment) outside the handler method, you ensure it runs once and is reused.

**Why it matters:** Lambda reuses execution environments across invocations and freezes them between calls, preserving static state. Initialization that runs during the `Init` phase is billed as part of the cold start, which occurs infrequently. The same code inside the handler runs on every invocation, making every call slower and more expensive.

* * *

### Use Provisioned Concurrency for Latency-Critical Paths

**What it is:** Provisioned Concurrency pre-initializes a specified number of execution environments, keeping them warm and ready to handle requests with no cold start. The environments are fully initialized — your static constructors have already run.

**Why it matters:** For synchronous, user-facing functions (e.g., API endpoints powering a mobile app), cold start latency can directly degrade user experience. Provisioned Concurrency eliminates cold starts entirely for those pre-warmed environments. It should not be used for async workers, background jobs, or event-processing functions, where latency is not user-visible and the added cost is unjustified.

* * *

### Lazy-Load Non-Critical Dependencies

**What it is:** For dependencies only needed in specific code paths, use `Lazy<T>` to defer their initialization until the first time that code path is actually executed, rather than initializing them unconditionally at startup.

**Why it matters:** Eagerly initializing every dependency at startup contributes to cold start duration and memory usage, even for code paths that may never be invoked in a given execution environment. Lazy loading ensures you only pay the initialization cost for dependencies that are actually used.

* * *

### Avoid Heavy DI Containers in the Hot Path

**What it is:** Reflection-based dependency injection frameworks (`Microsoft.Extensions.DependencyInjection` with assembly scanning, Autofac, etc.) perform extensive reflection during container construction. This conflicts directly with both Native AOT (which requires all types to be known at compile time) and cold start minimization goals.

**Why it matters:** Reflection-based DI scanning can add hundreds of milliseconds to your `Init` phase and is incompatible with Native AOT trimming. Preferred alternatives are manual wiring (fastest, recommended for simple functions) and source-generated DI (for more complex compositions).

* * *

## Execution & Compute Efficiency

### Use async/await Throughout — Avoid .Result and .Wait()

**What it is:** `async`/`await` enables non-blocking I/O in .NET — when a function awaits a network call, the thread is released to do other work. `.Result` and `.Wait()` are synchronous blocking calls that hold the thread until the operation completes.

**Why it matters:** Lambda bills for wall-clock time, not CPU time. A thread blocked on `.Result` or `.Wait()` holds the execution environment hostage while waiting for I/O, consuming billed duration even though it is doing nothing. It can also cause deadlocks in certain synchronization contexts.

* * *

### Reuse HttpClient and AWS SDK Clients Across Invocations

**What it is:** `HttpClient` and AWS SDK service clients (e.g., `AmazonDynamoDBClient`) are designed to be long-lived and thread-safe. They should be created once as static fields and reused across invocations, not constructed per-request.

**Why it matters:** Each `new HttpClient()` creates a new socket pool. Creating one per invocation rapidly exhausts available ports (the default ephemeral port range is ~30,000), causing socket exhaustion under load — a notoriously difficult production bug to diagnose. AWS SDK clients hold connection pools internally; re-creating them per invocation defeats connection reuse and adds DNS resolution overhead on every call.

* * *

### Set Appropriate Timeout

**What it is:** Lambda's timeout setting defines the maximum duration an invocation is allowed to run before it is forcibly terminated. It is configurable from 1 second to 15 minutes per function.

**Why it matters:** Lambda's default timeout for new functions is 3 seconds — too short for most real workloads. But setting it to the maximum (15 minutes) "just in case" means a function that hangs due to a slow downstream service will bill the full 15 minutes before being terminated. Calculate your timeout based on measured performance:

```plaintext
Timeout = (p99 measured duration) × 2 + network overhead margin
```

For example, if your function normally completes in 800ms and has a p99 of 1.2 seconds, set timeout to 3–5 seconds, not 15 minutes.

```yaml
MyFunction:
  Type: AWS::Serverless::Function
  Properties:
    Timeout: 10    # Seconds — not 900 (15 min)
```

* * *

### Use System.Text.Json over Newtonsoft.Json

**What it is:** `System.Text.Json` (STJ) is the built-in .NET JSON library introduced in .NET Core 3.1. It is allocation-friendly, supports `Span<T>` for zero-copy parsing, and is fully compatible with Native AOT via source generation. `Newtonsoft.Json` is the older, widely used third-party library that predates STJ.

**Why it matters:** `Newtonsoft.Json` relies on reflection, is incompatible with Native AOT, adds ~500 KB to your deployment package, and is measurably slower for typical Lambda payloads. STJ delivers better throughput, lower memory pressure, and AOT compatibility with no external dependency.

* * *

### Avoid Boxing and Unnecessary Allocations in Hot Paths

**What it is:** Boxing occurs when a value type (e.g., `int`, `struct`) is implicitly converted to `object`, causing a heap allocation. Unnecessary allocations include creating objects, strings, arrays, or closures that are immediately discarded after a single use.

**Why it matters:** Lambda functions that process thousands of events per second generate enormous GC pressure if they produce many short-lived heap objects per invocation. GC pauses directly extend billed duration and increase tail latency.

* * *

## Concurrency & Scaling

### Set Reserved Concurrency to Avoid Noisy-Neighbor Throttling

**What it is:** Reserved concurrency is a per-function setting that both guarantees a minimum concurrency allocation for a function and caps its maximum concurrency. By default, all Lambda functions in an AWS account share a regional concurrency pool (default: 1,000 concurrent executions).

**Why it matters:** Without reserved concurrency, a single function experiencing a traffic spike can consume the entire regional pool, throttling completely unrelated functions in the same account. Reserved concurrency prevents this by isolating a function's allocation from the shared pool.

* * *

### Configure SQS Batch Size and Batch Window

**What it is:** When Lambda polls an SQS queue, it can retrieve and process multiple messages in a single invocation (a batch). `BatchSize` controls the maximum number of messages per invocation. `MaximumBatchingWindowInSeconds` controls how long Lambda waits to fill the batch before invoking. `ReportBatchItemFailures` tells Lambda which specific messages in a batch failed, so only those are retried.

**Why it matters:** Larger batches mean fewer Lambda invocations, which directly reduces cost. Without `ReportBatchItemFailures`, if one message in a batch of 100 fails, all 100 are returned to the queue and reprocessed — a 100x amplification of failed work.

```yaml
Resources:
  MyFunction:
    Type: AWS::Serverless::Function
    Events:
      SQSTrigger:
        Type: SQS
        Properties:
          Queue: !GetAtt MyQueue.Arn
          BatchSize: 100
          MaximumBatchingWindowInSeconds: 5
          FunctionResponseTypes:
            - ReportBatchItemFailures
```

> Message visibility timeout must be greater than max function execution time.

* * *

### Use Event Filtering on SQS/DynamoDB/Kinesis Triggers

**What it is:** Lambda event source filtering lets you define a filter pattern at the AWS service level. Messages that don't match the filter are discarded by the service before they ever invoke your function.

**Why it matters:** You are not billed for filtered-out events, and your function doesn't need to contain logic to discard irrelevant messages. This reduces invocation count, cost, and code complexity.

```yaml
Events:
  SQSTrigger:
    Type: SQS
    Properties:
      Queue: !GetAtt MyQueue.Arn
      FilterCriteria:
        Filters:
          - Pattern: '{"body": {"eventType": ["ORDER_PLACED"]}}'
```

* * *

### Tune Maximum Concurrency on Event Source Mappings

**What it is:** The `ScalingConfig.MaximumConcurrency` property on an event source mapping caps how many concurrent Lambda executions can be processing from that specific source simultaneously, independent of the function's overall reserved concurrency.

**Why it matters:** Without this limit, a large SQS queue backlog can scale Lambda to hundreds of concurrent executions simultaneously. This can overwhelm downstream databases or APIs not designed for that level of parallelism, causing cascading failures.

```yaml
Events:
  SQSTrigger:
    Type: SQS
    Properties:
      Queue: !GetAtt MyQueue.Arn
      BatchSize: 10
      ScalingConfig:
        MaximumConcurrency: 10
```

* * *

## Networking & Integrations

### Avoid VPC Unless Strictly Necessary

**What it is:** Placing a Lambda function inside a VPC attaches an Elastic Network Interface (ENI) to the function's execution environment, giving it access to private VPC resources. This is required for accessing services that have no public endpoint, such as RDS or ElastiCache.

**Why it matters:** ENI attachment adds 100–500ms of cold start latency and introduces capacity constraints in subnets. Lambda functions outside a VPC still run in AWS-managed, isolated infrastructure with no public inbound access — the security benefit of VPC placement is often overstated. Services like DynamoDB, S3, SQS, and SNS are all accessible without a VPC (or via VPC endpoints if the function is already in one).

* * *

### Use VPC Endpoints for AWS Services

**What it is:** VPC Endpoints (Gateway endpoints for S3/DynamoDB, Interface endpoints for most other services) route traffic between your Lambda function and AWS services privately within the AWS network, bypassing the public internet and NAT Gateway.

**Why it matters:** Without VPC endpoints, traffic from a VPC-based Lambda to AWS services routes through a NAT Gateway, which charges $0.045 per GB of data processed. For a high-throughput function reading large S3 objects, this adds up quickly. VPC endpoints eliminate the NAT Gateway data charge (Interface endpoints have a small hourly fee, but it is offset by avoided NAT costs above roughly 10 GB/month).

* * *

### Enable Connection Pooling and Keep-Alive for HTTP

**What it is:** HTTP connection pooling reuses established TCP connections across multiple requests rather than opening a new connection for each one. In Lambda, `SocketsHttpHandler` manages this pool, and it persists across invocations as long as the execution environment stays warm.

**Why it matters:** Without connection reuse, each Lambda invocation (or each HTTP call within it) incurs TCP handshake and TLS negotiation overhead. Tuning pool settings prevents stale connections from causing errors while keeping connections alive long enough to benefit from reuse.

```csharp
private static readonly HttpClient HttpClient = new HttpClient(new SocketsHttpHandler
{
    PooledConnectionLifetime = TimeSpan.FromMinutes(15),
    PooledConnectionIdleTimeout = TimeSpan.FromMinutes(2),
    MaxConnectionsPerServer = 50,
    UseCookies = false
})
{
    Timeout = TimeSpan.FromSeconds(10)
};
```

* * *

### Configure AWS SDK Retry and Timeout

**What it is:** The AWS SDK has a built-in retry policy that automatically retries failed requests with exponential backoff. The default configuration retries up to 3 times. Both the retry count and per-attempt timeout can be configured on each SDK client.

**Why it matters:** For a Lambda function with a 5-second timeout calling a struggling DynamoDB table, the SDK's default aggressive retry policy may consume the entire timeout window retrying, billing you for the full duration and still returning an error. Setting explicit retry limits and per-call timeouts gives you control over this behavior.

* * *

## Observability & Cost Visibility

### Enable Lambda Insights

**What it is:** Lambda Insights is an enhanced CloudWatch monitoring feature that captures per-invocation metrics including memory used, CPU time, init duration, and network I/O. It is enabled by attaching the managed `LambdaInsightsExtension` Lambda layer to your function.

**Why it matters:** The default Lambda CloudWatch metrics (invocations, duration, errors) don't surface resource-level detail like memory pressure or CPU utilization. Lambda Insights fills that gap, making it possible to identify memory leaks, CPU-bound invocations, and slow init phases without adding custom instrumentation.

> **Cost note:** Lambda Insights charges for custom CloudWatch metrics (~$0.30 per metric per month) and additional log data. For high-traffic functions, filter what you emit.

* * *

### Use Structured Logging with Log Level Filtering

**What it is:** Structured logging emits log entries as JSON objects with consistent fields (timestamp, level, message, correlation IDs, etc.) rather than plain text strings. Log level filtering suppresses verbose logs (e.g., `Debug`, `Trace`) in production while keeping `Warning` and `Error` output.

**Why it matters:** Plain string logs are hard to query at scale. Structured JSON logs can be filtered and aggregated efficiently with CloudWatch Logs Insights. Log level control reduces log volume and CloudWatch ingestion costs in production.

> To accomplish this easily, use [Powertools for AWS Lambda (.NET)](https://docs.aws.amazon.com/powertools/dotnet/).

* * *

### Set CloudWatch Log Retention Policy

**What it is:** Each Lambda function automatically gets a CloudWatch Log Group. The default retention policy is "Never Expire," meaning logs accumulate indefinitely. You can configure a retention period (e.g., 14 days) via CloudFormation or the console.

**Why it matters:** CloudWatch Logs storage is billed at $0.03/GB. With "Never Expire," logs from high-volume functions accumulate indefinitely, and the bill grows silently. Explicitly setting a retention period caps storage costs and keeps log groups manageable.

```yaml
Resources:
  MyFunctionLogGroup:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: !Sub "/aws/lambda/${MyFunction}"
      RetentionInDays: 14
```

* * *

### Use X-Ray or OpenTelemetry for Distributed Tracing

**What it is:** AWS X-Ray is a distributed tracing service that instruments calls across Lambda invocations, AWS service calls (DynamoDB, S3, SQS), and outbound HTTP requests, producing service maps and per-segment latency breakdowns. OpenTelemetry is the vendor-neutral alternative that can export to X-Ray, Jaeger, or other backends.

**Why it matters:** Standard CloudWatch metrics tell you that a function is slow; distributed tracing tells you *which downstream call* is responsible. This is invaluable for identifying the specific DynamoDB query or external API call causing p99 latency spikes.

```yaml
Globals:
  Function:
    Tracing: Active
```

> To accomplish this easily, use [Powertools for AWS Lambda (.NET)](https://docs.aws.amazon.com/powertools/dotnet/).

* * *

### Tag All Lambda Functions with Cost Allocation Tags

**What it is:** AWS resource tags are key-value pairs attached to resources. Cost Allocation Tags are tags you activate in the Billing console, after which AWS Cost Explorer can group and filter Lambda spending by those tag values.

**Why it matters:** Without consistent tagging, Lambda costs in a shared account appear as a single line item, making it impossible to attribute spending to specific teams, features, or environments. Tags enable per-team or per-service cost accountability.

```yaml
Globals:
  Function:
    Tags:
      Team: payments
      Environment: production
      Service: order-processing
```

* * *

## Architecture & Design

### Keep Functions Single-Purpose

**What it is:** A single-purpose Lambda function handles one specific operation (e.g., process an order event, resize an image, send a notification) rather than branching across multiple unrelated operations based on input type.

**Why it matters:** A function that branches heavily based on input type ends up sized for its most demanding branch, wasting memory and compute for all other branches. Single-purpose functions are easier to size, tune independently, monitor, and debug.

* * *

### Consider Lambda URLs vs API Gateway

**What it is:** Lambda Function URLs are built-in HTTPS endpoints directly on a Lambda function, requiring no additional AWS service. API Gateway is a fully managed service that sits in front of Lambda and adds features like request validation, authorizers, usage plans, WAF integration, and stage variables.

**Why it matters:** For simple HTTP endpoints that don't need API Gateway features, Lambda Function URLs eliminate the $1.00 per million request charge that HTTP API Gateway adds, reducing cost to Lambda invocation charges only. The right choice depends on whether you need the additional API Gateway capabilities.

| Feature | API Gateway HTTP | Lambda URL |
| --- | --- | --- |
| Cost per million requests | $1.00 | $0 (Lambda invocation only) |
| CORS support | Yes | Yes |
| Auth | IAM, Cognito, Lambda authorizer | IAM, none |
| Custom domain | Yes | No (but works behind CloudFront) |
| WebSocket | Yes | No |

* * *

### Use Step Functions for Orchestration over Chained Lambdas

**What it is:** AWS Step Functions is a serverless orchestration service that coordinates multi-step workflows. The alternative — chaining Lambda functions by having one function synchronously invoke another and wait for its response — keeps the calling function's execution environment alive and billing during the entire wait.

**Why it matters:** Synchronously chaining Lambda functions doubles or triples your Lambda bill because the caller is billed for the full wait duration. Step Functions handles orchestration at the service level; your Lambda functions execute independently and are only billed for their own compute time.

* * *

## Security

### Apply Least-Privilege IAM Execution Roles

**What it is:** Every Lambda function runs under an IAM execution role. This role defines exactly which AWS API calls the function is permitted to make. Many teams default to broad managed policies like `AmazonDynamoDBFullAccess` or even `AdministratorAccess` during development and never revisit them.

**Why it matters:** If your function is compromised through a vulnerability in your code or a dependency, the attacker inherits every permission in that execution role. A function that can only call `dynamodb:GetItem` on one specific table does far less damage than one with write access to all tables in the account.

* * *

### Never Store Secrets in Environment Variables or Source Code

**What it is:** Lambda environment variables are convenient but are stored in plaintext in the function configuration, visible to anyone with `lambda:GetFunctionConfiguration` or `lambda:GetFunction` IAM access. The correct alternative is AWS Secrets Manager or AWS Systems Manager Parameter Store (SecureString), where secrets are fetched at runtime and access is controlled via IAM.

**Why it matters:** Database passwords, API keys, and signing secrets in environment variables or source code have been the root cause of numerous high-profile cloud breaches. The AWS Shared Responsibility Model makes secret storage entirely your responsibility.

* * *

### Enable AWS WAF on API Gateway or CloudFront

**What it is:** AWS WAF (Web Application Firewall) inspects HTTP requests before they reach your Lambda function, blocking common attack patterns (SQL injection, XSS, path traversal), known malicious IP ranges, and volumetric abuse. It is attached to API Gateway, CloudFront, or an Application Load Balancer.

**Why it matters:** Without WAF, a Lambda function exposed via API Gateway is reachable by anyone on the internet. Even a well-validated function can be overwhelmed by a flood of requests (costing money in Lambda invocations) or hit with sophisticated payloads designed to exploit dependencies.

* * *

### Use Resource-Based Policies to Restrict Who Can Invoke

**What it is:** A Lambda resource policy (also called a function policy) defines which AWS principals — accounts, services, or IAM roles — are allowed to call `lambda:InvokeFunction` on your function. It is separate from the function's execution role, which controls what the function can do.

**Why it matters:** Without a restrictive resource policy, an API Gateway misconfiguration or an accidental public URL can expose your function to arbitrary invocation from the internet, leading to unexpected charges and potential data exposure.

* * *

### Restrict Lambda Function URL Auth

**What it is:** Lambda Function URLs support two authentication modes: `AuthType: NONE` (publicly invocable by anyone with the URL) and `AuthType: AWS_IAM` (requires a valid AWS Signature Version 4 signed request). The `NONE` mode is occasionally used for public webhooks but is a security risk for most production functions.

**Why it matters:** `AuthType: NONE` means the URL is publicly accessible with no authentication. Anyone who discovers or guesses the URL can invoke your function at your expense and potentially exfiltrate or corrupt data. Use `AWS_IAM` for any function that is not intentionally public.

* * *

## Operations

### Configure Dead Letter Queues on Async Invocations

**What it is:** A Dead Letter Queue (DLQ) is an SQS queue or SNS topic configured to receive events that Lambda failed to process after exhausting its retry attempts. For asynchronous invocations (from SNS, S3 events, EventBridge, or direct async `InvokeFunction` calls), Lambda retries failed invocations twice by default before discarding the event.

**Why it matters:** Without a DLQ, a transient downstream failure (a throttled DynamoDB table, a network timeout) can cause permanent, silent data loss with no alert and no recovery path. Silent event loss is one of the hardest production bugs to detect.

```yaml
Resources:
  MyFunctionDLQ:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: my-function-dlq
      MessageRetentionPeriod: 1209600   # 14 days in seconds

  MyFunction:
    Type: AWS::Serverless::Function
    Properties:
      Runtime: dotnet10
      DeadLetterQueue:
        Type: SQS
        TargetArn: !GetAtt MyFunctionDLQ.Arn
      EventInvokeConfig:
        MaximumRetryAttempts: 2
        MaximumEventAgeInSeconds: 3600
```

> Always alarm on DLQ depth so a backlog is immediately visible.

* * *

### Use Lambda Destinations for Async Success and Failure Routing

**What it is:** Lambda Destinations are a more flexible evolution of DLQs. They let you route both successful and failed async invocations to an SQS queue, SNS topic, EventBridge bus, or another Lambda function, with the full original event and function response or error details included in the routed payload.

**Why it matters:** DLQs receive only the original input event on failure. Destinations receive the original event *plus* the function response or error details for both successes and failures, making post-processing, alerting, and conditional routing significantly richer. They also enable success-path routing, which DLQs cannot do.

* * *

### Alarm on Errors, Throttles, and Duration Breaches

**What it is:** Lambda publishes several CloudWatch metrics out of the box: `Errors`, `Throttles`, `Duration`, `ConcurrentExecutions`, and `IteratorAge` (for stream-based triggers). CloudWatch Alarms can monitor these metrics and trigger notifications or automated actions when thresholds are breached.

**Why it matters:** None of these metrics have alarms by default — you must create them explicitly. Without alarms, a function silently failing 10% of invocations, hitting account-level concurrency limits, or processing events that are hours behind will go undetected until a user or downstream system reports a problem.

```yaml
ErrorRateAlarm:
  Type: AWS::CloudWatch::Alarm
  Properties:
    AlarmName: !Sub "${MyFunction}-ErrorRate"
    Metrics:
      - Id: errors
        MetricStat:
          Metric:
            Namespace: AWS/Lambda
            MetricName: Errors
            Dimensions: [{ Name: FunctionName, Value: !Ref MyFunction }]
          Period: 60
          Stat: Sum
      - Id: invocations
        MetricStat:
          Metric:
            Namespace: AWS/Lambda
            MetricName: Invocations
            Dimensions: [{ Name: FunctionName, Value: !Ref MyFunction }]
          Period: 60
          Stat: Sum
      - Id: errorRate
        Expression: "errors / invocations * 100"
        Label: ErrorRate
    ComparisonOperator: GreaterThanThreshold
    Threshold: 1
    EvaluationPeriods: 2
    TreatMissingData: notBreaching
    AlarmActions: [!Ref OpsAlertTopic]

ThrottleAlarm:
  Type: AWS::CloudWatch::Alarm
  Properties:
    AlarmName: !Sub "${MyFunction}-Throttles"
    MetricName: Throttles
    Namespace: AWS/Lambda
    Dimensions: [{ Name: FunctionName, Value: !Ref MyFunction }]
    Statistic: Sum
    Period: 300
    EvaluationPeriods: 1
    Threshold: 1
    ComparisonOperator: GreaterThanOrEqualToThreshold
    AlarmActions: [!Ref OpsAlertTopic]

DurationAlarm:
  Type: AWS::CloudWatch::Alarm
  Properties:
    AlarmName: !Sub "${MyFunction}-DurationP99"
    MetricName: Duration
    Namespace: AWS/Lambda
    Dimensions: [{ Name: FunctionName, Value: !Ref MyFunction }]
    ExtendedStatistic: p99
    Period: 300
    EvaluationPeriods: 3
    Threshold: 8000   # 8 seconds if timeout is 10 seconds
    ComparisonOperator: GreaterThanThreshold
    AlarmActions: [!Ref OpsAlertTopic]
```

* * *

### Use Lambda Aliases and Traffic Shifting for Safe Deployments

**What it is:** Lambda aliases are named pointers to specific function versions (e.g., `live` → version 12). Traffic shifting, configured through AWS CodeDeploy, allows you to route a percentage of invocations to a new version while the majority still goes to the current stable version — a serverless canary or linear deployment.

**Why it matters:** Deploying a new function version directly to 100% of traffic has no rollback path faster than re-deploying the previous version, which takes 30–90 seconds. With canary or linear deployments, CodeDeploy monitors your CloudWatch alarms and automatically rolls back to the previous version the moment an alarm fires — often within 60 seconds of a bad deployment.

* * *

## Quick Reference

> **Must** = required for any production function regardless of workload. **Optional** = strongly advisable but context-dependent — the "When to apply" column clarifies when it becomes effectively mandatory.

| # | Practice | Priority | When to apply |
| --- | --- | --- | --- |
| **Runtime & Deployment** |  |  |  |
| 1 | Use latest LTS .NET on arm64 | Optional | New functions or runtime upgrades; skip if library incompatibility blocks migration |
| 2 | Minimize deployment package size | Must | Always |
| 3 | Use Native AOT compilation | Optional | New functions; skip if dependencies are AOT-incompatible |
| 4 | Use Lambda SnapStart | Optional | JIT-based functions where cold start is a problem and AOT is not feasible |
| **Memory & CPU Sizing** |  |  |  |
| 5 | Profile with Lambda Power Tuning | Must | Before any function goes to production; re-run after significant code changes |
| 6 | Don't under-allocate memory | Must | Always |
| 7 | Set memory based on actual measurements | Must | Always |
| **Cold Start Optimization** |  |  |  |
| 8 | Move heavy initialization outside the handler | Must | Always |
| 9 | Use Provisioned Concurrency | Optional | Synchronous user-facing functions with strict latency SLAs only |
| 10 | Lazy-load non-critical dependencies | Optional | Functions with multiple code paths that aren't always exercised |
| 11 | Avoid heavy DI containers in the hot path | Must | Always; especially mandatory when using Native AOT |
| **Execution & Compute Efficiency** |  |  |  |
| 12 | Use async/await — avoid .Result and .Wait() | Must | Always |
| 13 | Reuse HttpClient and AWS SDK clients | Must | Always |
| 14 | Set appropriate timeout | Must | Always |
| 15 | Use System.Text.Json over Newtonsoft.Json | Must | New functions; for existing functions, mandatory when targeting Native AOT |
| 16 | Avoid boxing and unnecessary allocations | Optional | High-throughput functions processing thousands of events per second |
| **Concurrency & Scaling** |  |  |  |
| 17 | Set reserved concurrency | Must | Any function in a shared account with other critical functions |
| 18 | Configure SQS batch size and ReportBatchItemFailures | Must | All SQS-triggered functions |
| 19 | Use event source filtering | Optional | When the function receives a mixed event stream and only acts on a subset |
| 20 | Tune maximum concurrency on event source mappings | Must | When the downstream target (DB, API) has a known parallelism limit |
| **Networking & Integrations** |  |  |  |
| 21 | Avoid VPC unless strictly necessary | Must | Always — only place in VPC when there is no alternative |
| 22 | Use VPC endpoints for AWS services | Must | Any VPC-attached function that calls AWS services |
| 23 | Enable connection pooling and keep-alive | Must | Always |
| 24 | Configure AWS SDK retry and timeout | Must | Always |
| **Observability & Cost Visibility** |  |  |  |
| 25 | Enable Lambda Insights | Optional | Functions where resource-level diagnostics (CPU, memory trend) are needed |
| 26 | Use structured logging with log level filtering | Must | Always |
| 27 | Set CloudWatch log retention policy | Must | Always |
| 28 | Use X-Ray or OpenTelemetry | Optional | Functions that call downstream services and where latency attribution matters |
| 29 | Tag all functions with cost allocation tags | Must | Any shared or multi-team AWS account |
| **Architecture & Design** |  |  |  |
| 30 | Keep functions single-purpose | Must | Always |
| 31 | Consider Lambda URLs vs API Gateway | Optional | Simple HTTP endpoints with no need for API Gateway features |
| 32 | Use Step Functions for orchestration | Must | Any workflow that chains Lambda calls synchronously |
| **Security** |  |  |  |
| 33 | Apply least-privilege IAM execution roles | Must | Always |
| 34 | Never store secrets in environment variables | Must | Always |
| 35 | Enable AWS WAF on API Gateway or CloudFront | Must | Any publicly exposed HTTP endpoint |
| 36 | Use resource-based policies to restrict invocation | Must | Always |
| 37 | Restrict Lambda Function URL auth | Must | Any Function URL not intentionally public |
| **Operations** |  |  |  |
| 38 | Configure Dead Letter Queues on async invocations | Must | All async-invoked functions |
| 39 | Use Lambda Destinations | Optional | When you need success-path routing or richer failure payloads beyond a basic DLQ |
| 40 | Alarm on errors, throttles, and duration | Must | Always |
| 41 | Use aliases and traffic shifting for deployments | Must | Any function where a bad deploy would have immediate user or data impact |

The current checklist is a starting point. Adapt it to your team's context, add items that reflect your specific compliance requirements or architectural patterns, and remove items that genuinely do not apply to your workload. The goal is not a perfect score — it is a shared, team-maintained definition of what a production-ready Lambda function looks like for your organization. Thanks, and happy coding