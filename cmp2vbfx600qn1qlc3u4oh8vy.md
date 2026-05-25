---
title: "AWS SQS Queue: Check List"
datePublished: 2026-05-12T16:51:42.818Z
cuid: cmp2vbfx600qn1qlc3u4oh8vy
slug: aws-sqs-queue-check-list
cover: https://cdn.hashnode.com/uploads/covers/625b81432f84e0d4fc33dca5/dd5ebde5-27b8-44ac-bc10-ab8450c8cf06.png
tags: aws, dotnet, aws-lambda, sqs

---

Amazon Simple Queue Service (SQS) is deceptively simple to get started with, but surprisingly easy to misconfigure in ways that cause silent message loss, runaway costs, duplicate processing storms, or throughput bottlenecks that only surface under load. A queue that "works in dev" and a queue that is production-ready are two very different things.

This guide covers eight areas of SQS excellence, each derived from real-world production patterns and common failure modes. It assumes you are comfortable with the basics of SQS and focuses on the *why* behind each practice as much as the *how*.

* * *

## Queue Design

### Choose the Right Queue Type: Standard vs FIFO

**What it is:** SQS offers two queue types. **Standard** queues offer maximum throughput (virtually unlimited TPS), at-least-once delivery, and best-effort ordering. **FIFO** queues guarantee exactly-once processing (deduplication within a 5-minute deduplication window) and strict ordering within a message group, but are capped at 300 API calls/second (or up to 70,000 with high-throughput mode in some regions).

**Why it matters:** Teams routinely default to Standard queues for everything, then bolt on deduplication logic in application code — or default to FIFO for everything and unknowingly cap their throughput at 300 TPS. Choose intentionally:

| Characteristic | Standard | FIFO |
| --- | --- | --- |
| Throughput | Unlimited | 300–70,000 TPS |
| Delivery guarantee | At-least-once | Exactly-once |
| Message ordering | Best-effort | Strict per group |
| Cost | Lower | ~10% higher |
| Right choice when... | High throughput, idempotent consumers | Ordering matters or duplicates are unacceptable |

> If you choose Standard queues, your consumers **must** be idempotent. Design for it explicitly — don't assume it.

* * *

### Set the Visibility Timeout Correctly

**What it is:** When a consumer receives a message, SQS hides it from other consumers for the **visibility timeout** duration. If the consumer does not delete the message within that window, SQS makes it visible again and another consumer (or the same one) will receive it. The default is 30 seconds.

**Why it matters:** A visibility timeout that is too short causes messages to be re-delivered while still being processed — leading to duplicate processing. A timeout that is too long means a consumer crash causes a long delay before recovery. The correct formula is:

```plaintext
Visibility Timeout = (p99 processing time) × 2 + buffer
```

For a function that normally completes in 10 seconds with a p99 of 25 seconds, set the visibility timeout to 60 seconds — not the default 30.

> When using Lambda as a consumer, the visibility timeout **must** be greater than the Lambda function's timeout. AWS will warn you if they match, but not if the queue timeout is only slightly larger.

```yaml
MyQueue:
  Type: AWS::SQS::Queue
  Properties:
    VisibilityTimeout: 60   # Must exceed consumer processing time
```

* * *

### Configure Message Retention Period Appropriately

**What it is:** SQS retains messages for a configurable period between 1 minute and 14 days. The default is 4 days. After the retention period expires, messages are permanently deleted whether they were processed or not.

**Why it matters:** A 4-day default sounds generous, but a consumer outage over a long weekend — combined with a DLQ not being configured — can cause silent, permanent message loss before anyone is back at a keyboard. Set retention to 14 days (the maximum) for business-critical queues so you have time to diagnose and recover.

```yaml
MyQueue:
  Type: AWS::SQS::Queue
  Properties:
    MessageRetentionPeriod: 1209600   # 14 days in seconds
```

> Apply the same 14-day retention to your Dead Letter Queue. DLQ messages accumulate *after* the main queue's retry attempts are exhausted — if the DLQ expires them before you triage, you lose the evidence.

* * *

### Use a Dead Letter Queue (DLQ)

**What it is:** A Dead Letter Queue is a separate SQS queue that receives messages after they have failed processing a configurable number of times (`maxReceiveCount`). When a message's receive count exceeds this threshold, SQS moves it to the DLQ automatically.

**Why it matters:** Without a DLQ, a poison-pill message — one that always causes your consumer to fail or crash — will loop indefinitely: receive, fail, become visible, receive, fail. It will block throughput (especially on FIFO queues), inflate costs, and produce a flood of error logs with no recovery path. The DLQ isolates the bad message and keeps the main queue moving.

```yaml
MyQueueDLQ:
  Type: AWS::SQS::Queue
  Properties:
    QueueName: my-queue-dlq
    MessageRetentionPeriod: 1209600   # 14 days

MyQueue:
  Type: AWS::SQS::Queue
  Properties:
    VisibilityTimeout: 60
    RedrivePolicy:
      deadLetterTargetArn: !GetAtt MyQueueDLQ.Arn
      maxReceiveCount: 3   # Move to DLQ after 3 failed attempts
```

> Set `maxReceiveCount` based on how many transient retries make sense for your workload. For idempotent consumers handling transient downstream failures, 3–5 is typical. Setting it to 1 means any single processing failure immediately sidelines the message.

* * *

### Tune Maximum Message Size and Use S3 for Large Payloads

**What it is:** SQS has a hard limit of 256 KB per message. For payloads that exceed this, the standard pattern is to store the payload in S3 and send a reference message containing the S3 object key.

**Why it matters:** Attempting to send a 300 KB payload fails with an exception at runtime. More subtly, even payloads *under* 256 KB that are unnecessarily large inflate costs (SQS charges per 64 KB chunk), slow down serialization, and consume more memory in your consumer. Design messages to carry only what is needed to identify and route the work — not the entire data blob.

> A well-designed SQS message is a *task descriptor*, not a *data transfer object*. Include IDs and event type; load the full record from the source of truth in the consumer.

* * *

## Message Design

### Design for Idempotency

**What it is:** An idempotent consumer produces the same result regardless of how many times it processes the same message. This is achieved through techniques like conditional writes (DynamoDB's `ConditionExpression`), database-level unique constraints, or tracking processed message IDs in a state store.

**Why it matters:** Standard queues guarantee at-least-once delivery — the same message can be delivered more than once. Even FIFO queues can deliver duplicates in rare failure scenarios. If your consumer is not idempotent, duplicate delivery causes duplicate side effects: double charges, duplicate records, or repeated emails. This is one of the most common and most expensive SQS production bugs.

* * *

### Include Correlation IDs in Messages

**What it is:** A correlation ID is a unique identifier (typically a UUID) that travels with a request through every system it touches — from the original producer through SQS, into the consumer, and onward to downstream services. It is typically stored in a message attribute rather than the message body.

**Why it matters:** Without correlation IDs, debugging a failed processing chain across a distributed system means manually matching log timestamps and guessing at relationships between events. With correlation IDs, a single value lets you pull every log line related to one request across every service in your stack in seconds.

```csharp
var sendRequest = new SendMessageRequest
{
    QueueUrl = _queueUrl,
    MessageBody = JsonSerializer.Serialize(payload),
    MessageAttributes = new Dictionary<string, MessageAttributeValue>
    {
        ["CorrelationId"] = new MessageAttributeValue
        {
            DataType = "String",
            StringValue = correlationId ?? Activity.Current?.TraceId.ToString() ?? Guid.NewGuid().ToString()
        },
        ["EventType"] = new MessageAttributeValue
        {
            DataType = "String",
            StringValue = "OrderPlaced"
        }
    }
};
```

* * *

### Use Message Attributes for Metadata

**What it is:** SQS supports up to 10 message attributes — typed key-value pairs that travel alongside the message body. Attributes can be read without deserializing the body and can be used for consumer-side filtering (when integrated with SNS) or routing logic.

**Why it matters:** Embedding routing metadata in the message body requires deserializing the entire payload before deciding what to do with the message. Message attributes surface that metadata cheaply — your consumer can branch on `EventType` or `TenantId` without touching the body. This is especially powerful with Lambda event source filtering, which inspects message attributes and body fields at the infrastructure level to avoid invoking the function at all for irrelevant messages.

* * *

### Set Delivery Delay Only When Genuinely Needed

**What it is:** SQS supports a per-queue or per-message delivery delay of 0–900 seconds. A delayed message is invisible to consumers for the delay period after it is sent.

**Why it matters:** Delivery delay is sometimes used as a poor man's scheduler ("wait 5 minutes before processing this"). This is fragile — if the queue is backed up, the message may sit even longer, and there is no way to cancel or reschedule a delayed message once sent. Use delay only for legitimate use cases such as rate-limiting a downstream system or building in a brief window for compensating events. For true scheduling, use EventBridge Scheduler or Step Functions instead.

* * *

## Producers

### Use SendMessageBatch to Reduce API Calls and Cost

**What it is:** The `SendMessageBatch` API sends up to 10 messages in a single API call. SQS pricing is per API call (per 64 KB chunk), regardless of whether the call sends 1 or 10 messages.

**Why it matters:** A producer sending 1,000 individual messages makes 1,000 API calls. The same producer using `SendMessageBatch` makes 100 API calls — a 10x cost reduction. At high volume, this adds up: 10 million messages/day via individual sends costs ~$4/day; batched, it costs ~$0.40/day.

* * *

### Handle Partial Failures in Batch Sends

**What it is:** Unlike most AWS APIs, `SendMessageBatch` does not throw an exception when some messages in the batch fail. It returns a `Failed` collection alongside the `Successful` collection. Callers that ignore `Failed` silently lose messages.

**Why it matters:** A dependency on a silent partial failure is a data loss bug waiting for the right conditions to trigger. Retry only the failed entries, not the entire batch, to avoid sending duplicates for the messages that succeeded.

* * *

## Consumers

### Always Use Long Polling

**What it is:** SQS supports two polling modes. **Short polling** queries a random subset of SQS servers and returns immediately, even if no messages are available. **Long polling** waits up to 20 seconds for messages to arrive before returning an empty response. Long polling is configured by setting `WaitTimeSeconds` to a value between 1 and 20.

**Why it matters:** Short polling frequently returns empty responses, each of which is a billable API call. For a consumer polling at 1-second intervals with no messages in the queue, that is 86,400 API calls per day — most of them empty. Long polling can dramatically reduce empty receives and polling costs and CPU overhead.

```csharp
var receiveRequest = new ReceiveMessageRequest
{
    QueueUrl = _queueUrl,
    MaxNumberOfMessages = 10,
    WaitTimeSeconds = 20,   // Long polling — always set this
    MessageAttributeNames = new List<string> { "All" }
};
```

> For Lambda event source mappings, AWS manages polling internally and always uses long polling. You do not need to configure this explicitly when using Lambda as a consumer.

* * *

### Process Messages in Batches

**What it is:** `ReceiveMessage` can return up to 10 messages per call. Processing messages individually (one receive call, one delete call, repeat) is the most expensive and least efficient consumption pattern.

**Why it matters:** Batch receive and batch delete (`DeleteMessageBatch`) reduce API call count by up to 10x. For Lambda consumers, larger batch sizes reduce the number of Lambda invocations and fixed per-invocation overhead, lowering cost and improving throughput.

```yaml
# SAM template — Lambda SQS trigger with batching
Events:
  SQSTrigger:
    Type: SQS
    Properties:
      Queue: !GetAtt MyQueue.Arn
      BatchSize: 10
      MaximumBatchingWindowInSeconds: 5    # Wait up to 5s to fill the batch
      FunctionResponseTypes:
        - ReportBatchItemFailures
```

* * *

### Handle Partial Batch Failures with ReportBatchItemFailures

**What it is:** By default, if any message in a Lambda batch fails, Lambda treats the entire batch as failed and re-delivers all messages — including those that were processed successfully. `ReportBatchItemFailures` lets the consumer report which specific message IDs failed, so only those are retried.

**Why it matters:** Without `ReportBatchItemFailures`, a single bad message in a batch of 100 causes all 100 to be reprocessed. For idempotent consumers, this is wasteful. For non-idempotent consumers, it is catastrophic. Always enable this, and always implement the response structure correctly:

```csharp
public async Task<SQSBatchResponse> FunctionHandler(SQSEvent sqsEvent, ILambdaContext context)
{
    var batchItemFailures = new List<SQSBatchResponse.BatchItemFailure>();

    foreach (var message in sqsEvent.Records)
    {
        try
        {
            await ProcessMessageAsync(message);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to process message {MessageId}", message.MessageId);
            batchItemFailures.Add(new SQSBatchResponse.BatchItemFailure
            {
                ItemIdentifier = message.MessageId
            });
        }
    }

    return new SQSBatchResponse { BatchItemFailures = batchItemFailures };
}
```

> Return an empty `BatchItemFailures` list when all messages succeed. Throwing an exception causes the entire batch to be retried.

* * *

### Delete Messages Only After Successful Processing

**What it is:** A message should only be deleted from SQS (or the receive acknowledged, for Lambda consumers) after the consumer has successfully completed all processing — including any writes to downstream systems.

**Why it matters:** Deleting a message before processing is complete (or before confirming downstream writes succeeded) creates a window for data loss. If the process crashes between the delete and the commit, the message is gone and the work was never done. SQS's visibility timeout exists precisely to handle this: if you don't delete the message, it becomes visible again and another consumer can pick it up.

* * *

### Tune MaximumConcurrency on Lambda Event Source Mappings

**What it is:** The `ScalingConfig.MaximumConcurrency` property caps how many concurrent Lambda executions can be processing from a specific SQS event source simultaneously, independently of the function's reserved concurrency.

**Why it matters:** A large SQS backlog can scale Lambda to hundreds of concurrent executions in seconds. If your downstream target — a relational database, a rate-limited external API — cannot absorb that level of concurrency, you will cause cascading failures. This setting is the correct throttle: it limits SQS-driven scale without affecting the function's ability to respond to other invocation sources.

```yaml
Events:
  SQSTrigger:
    Type: SQS
    Properties:
      Queue: !GetAtt MyQueue.Arn
      BatchSize: 10
      ScalingConfig:
        MaximumConcurrency: 5   # Never more than 5 concurrent consumers
```

* * *

## FIFO Queues

### Use Message Group IDs to Parallelize FIFO Throughput

**What it is:** In a FIFO queue, a Message Group ID defines an independent ordering stream. All messages with the same group ID are processed in order. Messages in different groups can be processed concurrently.

**Why it matters:** A FIFO queue with a single message group ID (or no group-level thinking) processes messages strictly serially — one at a time. For a customer-ordering system with millions of customers, using `customerId` as the group ID means each customer's orders are processed in order, while orders for different customers are processed in parallel. This is how FIFO queues achieve meaningful throughput.

```csharp
var sendRequest = new SendMessageRequest
{
    QueueUrl = _fifoQueueUrl,
    MessageBody = JsonSerializer.Serialize(orderEvent),
    MessageGroupId = customerId,            // Parallelism at the customer level
    MessageDeduplicationId = orderId        // Unique per message
};
```

* * *

### Configure Deduplication Correctly

**What it is:** FIFO queues deduplicate messages within a 5-minute window. Deduplication can be content-based (SQS hashes the message body) or explicit (you provide a `MessageDeduplicationId`). Content-based deduplication is enabled at the queue level; explicit deduplication IDs are set per message.

**Why it matters:** Content-based deduplication silently drops any two messages with identical bodies within 5 minutes — including legitimate messages that happen to be identical (e.g., two separate `OrderCancelled` events with the same item). Use explicit deduplication IDs (tied to the business event ID, not the content) unless you are certain identical bodies always represent duplicate events.

```yaml
MyFifoQueue:
  Type: AWS::SQS::Queue
  Properties:
    QueueName: my-queue.fifo          # .fifo suffix is required
    FifoQueue: true
    ContentBasedDeduplication: false  # Use explicit IDs instead
    VisibilityTimeout: 60
```

* * *

## Security

### Apply Least-Privilege IAM Policies

**What it is:** IAM policies control which principals can perform which actions on which SQS queues. Producers need `sqs:SendMessage`. Consumers need `sqs:ReceiveMessage`, `sqs:DeleteMessage`, and `sqs:GetQueueAttributes`. Admin operations (`sqs:CreateQueue`, `sqs:DeleteQueue`) should be reserved for infrastructure automation roles, never application roles.

**Why it matters:** A Lambda function with `sqs:*` on `*` can delete your queues, purge messages, or change queue attributes. The blast radius of a compromised or misconfigured function is proportional to its permissions. Scope each role to the exact actions and queue ARNs it needs.

```yaml
MyFunctionRole:
  Type: AWS::IAM::Role
  Properties:
    Policies:
      - PolicyName: SQSConsumerPolicy
        PolicyDocument:
          Statement:
            - Effect: Allow
              Action:
                - sqs:ReceiveMessage
                - sqs:DeleteMessage
                - sqs:GetQueueAttributes
              Resource: !GetAtt MyQueue.Arn   # Specific queue, not *
```

* * *

### Enable Server-Side Encryption (SSE)

**What it is:** SQS supports two SSE options: **SSE-SQS** (AWS-managed keys, no additional cost) and **SSE-KMS** (customer-managed keys). Both encrypt messages at rest.

**Why it matters:** Messages in SQS may contain PII, financial data, or internal business events. Without SSE, that data is stored in plaintext, accessible to anyone with sufficient AWS account-level access. SSE-SQS is free and zero-configuration — there is no reason not to enable it. Use SSE-KMS only when your compliance requirements demand customer-managed key control or key rotation auditability.

```yaml
MyQueue:
  Type: AWS::SQS::Queue
  Properties:
    SqsManagedSseEnabled: true   # SSE-SQS — free, always on
```

> If you use SSE-KMS and a Lambda consumer, the Lambda execution role must also have `kms:Decrypt` permission on the key, or message receipt will fail with an opaque permissions error.

* * *

### Use VPC Endpoints for SQS

**What it is:** An SQS VPC Interface Endpoint routes traffic between your VPC resources (EC2, ECS, Lambda in VPC) and SQS privately within the AWS network, without traversing the public internet or requiring a NAT Gateway.

**Why it matters:** Without a VPC endpoint, a VPC-attached Lambda or ECS task calling SQS must route through a NAT Gateway, incurring $0.045/GB data processing charges. For a high-throughput consumer processing thousands of messages per second, this adds up. VPC endpoints also eliminate the need to allow outbound internet access for SQS traffic, reducing your attack surface.

* * *

### Restrict Queue Access with Resource-Based Policies

**What it is:** An SQS queue policy (resource-based policy) defines which AWS accounts, IAM principals, or services are allowed to interact with the queue — independently of the caller's IAM identity policy. It is the primary mechanism for cross-account access and for restricting which services can publish to a queue.

**Why it matters:** Without a restrictive queue policy, any principal in your AWS account with `sqs:SendMessage` in their IAM policy can send to your queue. If your queue receives events from SNS, scope the `aws:SourceArn` condition to the specific SNS topic — not `*` — to prevent other SNS topics from publishing to your queue in the event of a misconfiguration.

```yaml
MyQueuePolicy:
  Type: AWS::SQS::QueuePolicy
  Properties:
    Queues:
      - !Ref MyQueue
    PolicyDocument:
      Statement:
        - Effect: Allow
          Principal:
            Service: sns.amazonaws.com
          Action: sqs:SendMessage
          Resource: !GetAtt MyQueue.Arn
          Condition:
            ArnEquals:
              aws:SourceArn: !Ref MySnsTopic   # Scope to specific topic
```

* * *

## Observability & Cost

### Monitor the Right CloudWatch Metrics

**What it is:** SQS publishes several CloudWatch metrics out of the box. The most operationally important are:

| Metric | What it tells you |
| --- | --- |
| `ApproximateNumberOfMessagesVisible` | Current depth of the queue — growing = consumers falling behind |
| `ApproximateAgeOfOldestMessage` | How stale the oldest unprocessed message is |
| `NumberOfMessagesSent` | Producer throughput |
| `NumberOfMessagesDeleted` | Consumer throughput |
| `ApproximateNumberOfMessagesNotVisible` | Messages currently in flight (being processed) |
| `NumberOfEmptyReceives` | Frequency of empty polls — high = long polling not enabled |

**Why it matters:** None of these metrics have alarms by default. `ApproximateAgeOfOldestMessage` is the single most important metric for detecting consumer lag — a queue that is growing or whose oldest message is hours old is silently accumulating backlog while your downstream systems fall further behind.

* * *

### Set Alarms on Queue Depth and Message Age

**What it is:** CloudWatch Alarms can trigger notifications (SNS, PagerDuty, Slack via EventBridge) when queue metrics breach thresholds. At minimum, alarm on `ApproximateAgeOfOldestMessage` and `ApproximateNumberOfMessagesVisible` on your DLQ.

**Why it matters:** A DLQ that silently fills up is an invisible data loss event in progress. An alarm on DLQ depth greater than 0 fires the moment the first message is dead-lettered, giving you immediate visibility before the situation compounds.

```yaml
DLQDepthAlarm:
  Type: AWS::CloudWatch::Alarm
  Properties:
    AlarmName: !Sub "${MyQueueDLQ}-Depth"
    MetricName: ApproximateNumberOfMessagesVisible
    Namespace: AWS/SQS
    Dimensions:
      - Name: QueueName
        Value: !GetAtt MyQueueDLQ.QueueName
    Statistic: Maximum
    Period: 60
    EvaluationPeriods: 1
    Threshold: 0
    ComparisonOperator: GreaterThanThreshold
    AlarmActions: [!Ref OpsAlertTopic]

QueueAgeAlarm:
  Type: AWS::CloudWatch::Alarm
  Properties:
    AlarmName: !Sub "${MyQueue}-MessageAge"
    MetricName: ApproximateAgeOfOldestMessage
    Namespace: AWS/SQS
    Dimensions:
      - Name: QueueName
        Value: !GetAtt MyQueue.QueueName
    Statistic: Maximum
    Period: 300
    EvaluationPeriods: 2
    Threshold: 300   # Alert if oldest message is over 5 minutes old
    ComparisonOperator: GreaterThanThreshold
    AlarmActions: [!Ref OpsAlertTopic]
```

* * *

### Tag Queues for Cost Allocation

**What it is:** AWS resource tags applied to SQS queues can be activated as Cost Allocation Tags in the Billing console, enabling AWS Cost Explorer to break down SQS spending by team, service, or environment.

**Why it matters:** SQS costs in a shared account appear as a single line item without tagging. High-volume queues can drive surprising costs (polling, storage, data transfer for KMS). Tags make per-team or per-service SQS cost accountability possible.

```yaml
MyQueue:
  Type: AWS::SQS::Queue
  Properties:
    Tags:
      - Key: Team
        Value: payments
      - Key: Environment
        Value: production
      - Key: Service
        Value: order-processing
```

* * *

## Operations

### Define Queues with Infrastructure as Code

**What it is:** SQS queues, DLQs, queue policies, alarms, and subscriptions should all be defined in CloudFormation, SAM, CDK, or Terraform — not created manually through the console. The queue URL and ARN should be passed to consuming services as configuration, never hardcoded.

**Why it matters:** Manually created queues accumulate configuration drift over time, have no change history, and cannot be reliably recreated after an accidental deletion or disaster recovery event. IaC also makes it trivial to enforce consistent settings (retention period, SSE, DLQ) across all queues by using shared templates or constructs.

* * *

### Have a Reprocessing Strategy for DLQ Messages

**What it is:** A Dead Letter Queue is only useful if you have a defined, tested process for triaging and reprocessing the messages in it. AWS SQS supports [DLQ redrive](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html#sqs-dead-letter-queues-redrive) — moving messages back to the source queue after a fix — via the console, CLI, or API.

**Why it matters:** Teams that configure a DLQ but never define a reprocessing runbook end up with a DLQ that fills up, triggers alarms, but causes no action because no one knows how to safely replay the messages. Document and test the runbook before you need it.

> Before redriving DLQ messages, make sure the root cause is fixed. Redriving without fixing the underlying bug just refills the DLQ.

* * *

### Handle Poison Pill Messages Explicitly

**What it is:** A poison pill message is a message that reliably causes consumer failure — typically due to malformed content, an unexpected schema, or data that triggers an unhandled edge case. Without a DLQ (or with `maxReceiveCount` set too high), poison pills loop indefinitely and block queue processing, especially on FIFO queues.

**Why it matters:** On a FIFO queue, a poison pill in a message group blocks every subsequent message in that group until the bad message is resolved. Even on Standard queues, a poison pill drains concurrency and generates continuous error noise. Configure `maxReceiveCount` low enough (3–5) that poison pills are sidelined quickly, and always alarm on DLQ depth so they are surfaced immediately.

* * *

## Quick Reference

> **Must** = required for any production queue regardless of workload. **Optional** = strongly advisable but context-dependent.

| # | Practice | Priority | When to apply |
| --- | --- | --- | --- |
| **Queue Design** |  |  |  |
| 1 | Choose Standard vs FIFO intentionally | Must | Always — never default without considering ordering and throughput needs |
| 2 | Set visibility timeout correctly | Must | Always — must exceed p99 consumer processing time |
| 3 | Set message retention to 14 days | Must | All business-critical queues |
| 4 | Configure a Dead Letter Queue | Must | Always |
| 5 | Use S3 for payloads > 64 KB | Must | Any queue where payload size is variable or large |
| **Message Design** |  |  |  |
| 6 | Design consumers for idempotency | Must | Always — mandatory for Standard queues, best practice for FIFO |
| 7 | Include correlation IDs | Must | Always |
| 8 | Use message attributes for metadata | Optional | When consumers need to route or filter without deserializing the body |
| 9 | Avoid delivery delay as a scheduler | Must | Always — use EventBridge Scheduler for real scheduling needs |
| **Producers** |  |  |  |
| 10 | Use SendMessageBatch | Must | Always for high-volume producers |
| 11 | Handle partial batch send failures | Must | Any code that calls SendMessageBatch |
| **Consumers** |  |  |  |
| 12 | Enable long polling (WaitTimeSeconds = 20) | Must | All non-Lambda consumers |
| 13 | Process messages in batches | Must | Always |
| 14 | Use ReportBatchItemFailures with Lambda | Must | All Lambda SQS consumers |
| 15 | Delete messages only after success | Must | Always |
| 16 | Cap MaximumConcurrency on Lambda ESM | Must | Any consumer where the downstream target has concurrency limits |
| **FIFO Queues** |  |  |  |
| 17 | Use business-level Message Group IDs | Must | All FIFO queue producers |
| 18 | Use explicit MessageDeduplicationId | Must | Unless identical bodies always mean duplicate events |
| **Security** |  |  |  |
| 19 | Apply least-privilege IAM per role | Must | Always |
| 20 | Enable Server-Side Encryption (SSE-SQS or SSE-KMS) | Must | Always — SSE-SQS is free; use SSE-KMS only when compliance requires customer-managed key control |
| 21 | Use VPC endpoints for SQS | Optional | Any VPC-attached consumer that calls SQS |
| 22 | Restrict queue with resource-based policy | Must | Any queue receiving cross-account or cross-service messages |
| **Observability & Cost** |  |  |  |
| 23 | Monitor the key CloudWatch metrics | Must | Always — at minimum ApproximateAgeOfOldestMessage and queue depth |
| 24 | Set alarms on queue depth, message age, and DLQ | Must | All business-critical queues — alarm on DLQ depth > 0 always |
| 25 | Tag queues with cost allocation tags | Must | Any shared or multi-team AWS account |
| **Operations** |  |  |  |
| 26 | Define queues in IaC | Must | Always |
| 27 | Document and test DLQ reprocessing runbook | Optional | Before going to production |
| 28 | Tune maxReceiveCount to isolate poison pills quickly | Must | Always |

The current checklist is a starting point. Adapt it to your team's context, add items that reflect your specific compliance requirements or architectural patterns, and remove items that genuinely do not apply to your workload. The goal is not a perfect score — it is a shared, team-maintained definition of what a production-ready SQS queue looks like for your organization. Thanks, and happy coding.