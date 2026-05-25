---
title: "AWS SNS Topic: Check List"
datePublished: 2026-05-16T14:08:02.672Z
cuid: cmp8f8dba000a1shxhu67122k
slug: aws-sns-topic-check-list
cover: https://cdn.hashnode.com/uploads/covers/625b81432f84e0d4fc33dca5/53b037cf-5407-4dfc-91aa-0d05021cc08a.png
tags: aws, dotnet, aws-lambda, sns

---

Amazon Simple Notification Service (SNS) looks straightforward on the surface — publish a message, deliver it to subscribers — but the gap between a topic that "works in dev" and one that is production-ready is surprisingly wide. Silent delivery failures, over-notification storms, runaway costs from unfiltered fan-out, and security misconfigurations all hide behind an API that never throws an obvious error.

This guide covers seven areas of SNS excellence, each derived from real-world production patterns and common failure modes. It assumes you are comfortable with the basics of SNS and focuses on the *why* behind each practice as much as the *how*.

* * *

## Topic Design

### Choose the Right Topic Type: Standard vs FIFO

**What it is:** SNS offers two topic types: **Standard** and **FIFO**.

**Standard** topics are designed for maximum throughput and broad compatibility across subscriber types. They provide at-least-once delivery and best-effort ordering, making them suitable for high-volume event distribution and fan-out architectures.

**FIFO** topics are designed for workloads where message ordering and deduplication are critical. They provide ordered delivery per message group and support deduplication within a 5-minute deduplication window using either content-based or explicit deduplication IDs. FIFO topics support FIFO SQS queues and Lambda subscribers, but not the full range of endpoint types available to Standard topics (such as email, SMS, or mobile push).

FIFO throughput is lower than Standard throughput, but AWS supports high-throughput FIFO mode, where throughput scales significantly when messages are distributed across multiple message groups and batching is used.

**Why it matters:** Teams often default to Standard topics and later attempt to reconstruct ordering guarantees in application code — a difficult and error-prone problem in distributed systems. Conversely, teams sometimes choose FIFO topics without understanding the operational trade-offs around throughput, subscriber compatibility, and message group design

| Characteristic | Standard | FIFO |
| --- | --- | --- |
| Throughput | Very high / effectively unlimited | Lower than Standard; scalable with high-throughput FIFO and batching |
| Delivery guarantee | At-least-once | Ordered delivery with deduplication support |
| Message ordering | Best-effort | Strict per message group |
| Supported subscribers | SQS, Lambda, HTTP/S, email, SMS, mobile push | FIFO SQS, Lambda |
| Right choice when... | High fan-out, heterogeneous subscribers, maximum scalability | Ordering and duplicate prevention are critical |

> Even with FIFO topics, downstream consumers should still be designed to be idempotent. Deduplication guarantees apply only within the deduplication window and do not eliminate all duplicate-processing scenarios across distributed systems.

> If you choose Standard topics, assume duplicate delivery can occur during normal operation — not just during failures or retries. Consumer idempotency is a production requirement, not an optional optimization.

* * *

### Define Topic Granularity Intentionally

**What it is:** Topic granularity is the decision of how broadly or narrowly a single SNS topic is scoped. You can publish all events from an entire system to one topic, one topic per aggregate type (e.g., `order-events`), or one topic per specific event type (e.g., `order-placed`, `order-cancelled`).

**Why it matters:** A "God topic" that receives every event type from every service creates three problems: consumers must subscribe to a firehose and filter locally; subscription filter policies become complex and fragile; and the topic becomes a hidden coupling point between every producer and every consumer in the system. Conversely, a topic per event type creates operational overhead with hundreds of nearly identical resources.

The right model is typically one topic per aggregate or bounded context (e.g., `orders-topic`, `inventory-topic`). Consumers use subscription filter policies to receive only the event types they care about.

> Topic proliferation is easier to recover from than a God topic. You can always merge two topics later. Splitting a God topic requires coordinating every producer and consumer simultaneously.

* * *

### Understand the 256 KB Payload Limit and Plan for Large Messages

**What it is:** SNS has a hard limit of 256 KB per published message. For larger payloads, the standard pattern is to store the data in S3 and publish a reference message containing the S3 object key (the "claim check" pattern). The AWS SNS Extended Client Library for Java and other languages automates this transparently.

**Why it matters:** A `PublishRequest` for a payload exceeding 256 KB throws an `InvalidParameterException` at runtime — there is no graceful degradation. More subtly, even messages *under* 256 KB that carry full data blobs inflate costs (SNS charges per 64 KB chunk) and create unnecessarily fat events. SNS messages should be *event notifications*, not data transfer objects.

```yaml
# Lambda consumer reading the claim-check reference
# The S3 key in the message body points to the full payload
{
  "eventType": "ReportGenerated",
  "correlationId": "abc-123",
  "payloadBucket": "my-events-bucket",
  "payloadKey": "reports/2026/05/abc-123.json"
}
```

> A well-designed SNS message carries enough context to route, log, and act on — not the full data record. Load the full payload from the source of truth in the consumer.

* * *

## Message Design

### Use an Event Envelope

**What it is:** An event envelope is a standard outer structure that wraps every message published to SNS, regardless of the event type. It typically contains metadata fields — event type, event version, source service, timestamp, and correlation ID — in a consistent, predictable schema.

**Why it matters:** Without an envelope, every producer invents its own message shape. Consumers must handle N different schemas, logging pipelines cannot extract common metadata without per-event-type parsing, and schema evolution (adding a field to "OrderPlaced") requires coordinating every consumer. An envelope makes all of that uniform at zero cost to the producer.

```csharp
public record EventEnvelope<T>
{
    public string EventId { get; init; } = Guid.NewGuid().ToString();
    public string EventType { get; init; } = typeof(T).Name;
    public string Source { get; init; } = string.Empty;
    public string Version { get; init; } = "1.0";
    public DateTime OccurredAt { get; init; } = DateTime.UtcNow;
    public string CorrelationId { get; init; } = string.Empty;
    public T Payload { get; init; } = default!;
}

var envelope = new EventEnvelope<OrderPlaced>
{
    Source = "order-service",
    CorrelationId = Activity.Current?.TraceId.ToString() ?? Guid.NewGuid().ToString(),
    Payload = new OrderPlaced { OrderId = orderId, CustomerId = customerId }
};

await snsClient.PublishAsync(new PublishRequest
{
    TopicArn = _topicArn,
    Message = JsonSerializer.Serialize(envelope),
    MessageAttributes = new Dictionary<string, MessageAttributeValue>
    {
        ["EventType"] = new MessageAttributeValue
        {
            DataType = "String",
            StringValue = "OrderPlaced"
        }
    }
});
```

* * *

### Include Correlation IDs in Every Message

**What it is:** A correlation ID is a unique identifier that travels with a request through every system it touches — from the originating API call through SNS, into every subscriber, and onward to downstream services. It is stored both in the message attributes (for filtering and routing) and in the event envelope (for body-level logging).

**Why it matters:** Without correlation IDs, debugging a failed event processing chain across a fan-out topology means manually matching log timestamps across multiple services and guessing at causality. With correlation IDs, one value pulls every log line related to one request across every subscriber in seconds. In a fan-out scenario where one SNS publish triggers five Lambda functions, the correlation ID is the only thread connecting them.

* * *

### Use Message Attributes for Filtering Metadata — Not Body Fields

**What it is:** SNS supports message attributes as typed key-value metadata attached to a message. In practice, the total message size (256 KB), including attributes, is the primary constraint. Subscription filter policies can evaluate these attributes to decide whether a subscriber receives a given message. Attributes are readable without deserializing the body.

**Why it matters:** SNS subscription filter policies can filter on message attributes or on the top-level keys of a JSON message body. Filtering on body keys requires raw message delivery to be disabled and forces SNS to parse the JSON body for every delivery attempt. Filtering on message attributes is cheaper, faster, and decouples the subscription routing logic from the message body schema. If your routing metadata could ever change its name or nest deeper in the body, message attributes isolate subscribers from that change.

```csharp
MessageAttributes = new Dictionary<string, MessageAttributeValue>
{
    ["EventType"] = new MessageAttributeValue
    {
        DataType = "String",
        StringValue = "OrderPlaced"
    },
    ["Region"] = new MessageAttributeValue
    {
        DataType = "String",
        StringValue = "US-EAST"
    },
    ["Priority"] = new MessageAttributeValue
    {
        DataType = "Number",
        StringValue = "1"
    }
}
```

* * *

### Design for Schema Evolution

**What it is:** Schema evolution is the practice of changing event schemas (adding fields, deprecating fields, splitting events) without breaking existing subscribers. The two main strategies are: additive-only changes with a version field in the envelope, or multiple parallel event versions published simultaneously.

**Why it matters:** SNS fan-out means a single schema change can break N subscribers simultaneously — and in a micro-services architecture, those subscribers are often owned by different teams with different deployment cadences. Publishing both `OrderPlaced/v1` and `OrderPlaced/v2` in parallel allows teams to migrate on their own schedule. Consumers that only read fields they know about and ignore unknown fields (Postel's Law) tolerate additive changes without any coordination.

> Never remove or rename a message attribute used by a subscription filter policy without auditing all subscriptions first. A filter that no longer matches silently drops all messages for that subscriber.

* * *

## Publishers

### Use PublishBatch to Reduce API Calls and Cost

**What it is:** The `PublishBatch` API sends up to 10 messages to the same topic in a single API call. SNS pricing is per API request (per 64 KB chunk), regardless of whether the call publishes 1 or 10 messages.

**Why it matters:** A producer publishing 1,000 individual events makes 1,000 API calls. The same producer using `PublishBatch` makes 100 calls — a 10x reduction. For an event-driven system publishing millions of events per day, batching can significantly reduce publisher-side API request overhead and may reduce SNS publish costs for high-volume workloads. Batching also reduces the number of network round trips, improving throughput for high-volume producers.

```csharp
var batchEntries = messages.Select((msg, index) => new PublishBatchRequestEntry
{
    Id = index.ToString(),
    Message = JsonSerializer.Serialize(msg),
    MessageAttributes = new Dictionary<string, MessageAttributeValue>
    {
        ["EventType"] = new MessageAttributeValue
        {
            DataType = "String",
            StringValue = msg.EventType
        }
    }
}).ToList();

var response = await snsClient.PublishBatchAsync(new PublishBatchRequest
{
    TopicArn = _topicArn,
    PublishBatchRequestEntries = batchEntries
});
```

* * *

### Handle Partial Failures in Batch Publishes

**What it is:** Like `SQS.SendMessageBatch`, `SNS.PublishBatch` does not throw an exception when individual messages in the batch fail. It returns a `Failed` collection alongside the `Successful` collection. Callers that ignore `Failed` silently lose events.

**Why it matters:** Partial batch failures are easy to miss in testing because they surface only under specific conditions — throttling, oversized payloads, transient network errors — that rarely appear in low-volume dev environments. Always inspect the `Failed` collection, log the `Code` and `Message` for each failure, and retry only the failed entries (not the whole batch) to avoid duplicates for the entries that succeeded.

```csharp
var response = await snsClient.PublishBatchAsync(request);

foreach (var failure in response.Failed)
{
    _logger.LogError(
        "SNS publish failed for message {Id}: [{Code}] {Message}",
        failure.Id, failure.Code, failure.Message);

    if (failure.SenderFault == false) // Throttling/server error — retryable
    {
        retryQueue.Add(batchEntries.First(e => e.Id == failure.Id));
    }
    // SenderFault == true means the message itself is invalid — do not retry
}
```

* * *

### Use Message Group IDs and Deduplication IDs for FIFO Topics

**What it is:** FIFO topics require a `MessageGroupId` per publish — this defines the ordering stream, analogous to a partition key. All messages with the same group ID are delivered in order. A `MessageDeduplicationId` (explicit or content-based) prevents duplicate delivery within the 5-minute deduplication window.

**Why it matters:** A FIFO topic where every message uses the same group ID processes messages strictly serially — effectively one at a time. For a system tracking events by customer, using `customerId` as the group ID means each customer's events are ordered while events for different customers are processed in parallel. Use explicit `MessageDeduplicationId` values tied to a business key (not content hash) to avoid silently dropping two legitimate identical events within the same deduplication window.

```csharp
await snsClient.PublishAsync(new PublishRequest
{
    TopicArn = _fifoTopicArn,
    Message = JsonSerializer.Serialize(envelope),
    MessageGroupId = customerId,
    MessageDeduplicationId = eventId   // Unique business key, not content hash
});
```

* * *

## Subscriptions

### Use Subscription Filter Policies to Reduce Unnecessary Deliveries

**What it is:** A subscription filter policy is a JSON document attached to an SNS subscription that specifies which messages should be delivered to that subscriber. Messages that do not match the policy are silently dropped at the SNS layer — the subscriber is never invoked.

**Why it matters:** Without filter policies, every subscriber receives every message published to the topic, regardless of relevance. In a fan-out topology with 10 subscribers on a single topic, publishing 1 million events means 10 million delivery attempts — most of which result in Lambda functions that immediately discard the message after deserializing it. Filter policies eliminate this wasted work at the infrastructure level, reducing Lambda invocations, SQS delivery counts, and downstream processing costs.

```yaml
MySubscription:
  Type: AWS::SNS::Subscription
  Properties:
    TopicArn: !Ref MyTopic
    Protocol: sqs
    Endpoint: !GetAtt MyQueue.Arn
    FilterPolicy:
      EventType:
        - OrderPlaced
        - OrderCancelled
    FilterPolicyScope: MessageAttributes   # Filter on attributes, not body
```

> A filter policy that matches *all* possible values of an attribute is equivalent to no filter — but it adds latency and complexity. Only attach a filter policy when you genuinely want a subset of messages.

* * *

### Enable Raw Message Delivery for SQS Subscribers

**What it is:** By default, SNS wraps every message in an SNS envelope before delivering to SQS — adding metadata fields like `MessageId`, `TopicArn`, `Timestamp`, and `UnsubscribeURL` around your actual payload. With raw message delivery enabled, SNS delivers only your original message body, without wrapping.

**Why it matters:** The default SNS envelope forces every SQS consumer to unwrap the SNS structure before accessing the payload — adding boilerplate to every consumer and coupling them to the SNS delivery format. With raw message delivery, the consumer sees exactly what was published, making it easier to reuse consumer code across direct SQS sends and SNS-triggered deliveries.

```yaml
MySubscription:
  Type: AWS::SNS::Subscription
  Properties:
    TopicArn: !Ref MyTopic
    Protocol: sqs
    Endpoint: !GetAtt MyQueue.Arn
    RawMessageDelivery: true   # Consumer receives the original message body
```

> Raw message delivery **cannot** be used with SNS subscription filter policies that filter on message body fields. If you enable both, filtering falls back to attribute-only matching. Use message attributes for filtering metadata (as covered in Message Design) and raw delivery works correctly.

* * *

### Configure a Dead Letter Queue for Every Subscription

**What it is:** An SNS subscription DLQ is an SQS queue that receives messages when SNS exhausts all delivery retry attempts to a subscriber. This is configured at the subscription level — not the topic level — because delivery failures are subscriber-specific (a healthy Lambda subscriber and a failing HTTP endpoint share the same topic, but only the HTTP endpoint's failures should land in a DLQ).

**Why it matters:** Without a subscription DLQ, messages that fail delivery after all retries are permanently and silently lost. SNS will retry HTTP/S endpoints for up to 23 days using exponential backoff, but SQS and Lambda subscribers have a much shorter retry window. A subscription DLQ captures those failures and gives you a chance to investigate and replay.

```yaml
MySubscription:
  Type: AWS::SNS::Subscription
  Properties:
    TopicArn: !Ref MyTopic
    Protocol: sqs
    Endpoint: !GetAtt MyQueue.Arn
    RedrivePolicy:
      deadLetterTargetArn: !GetAtt SubscriptionDLQ.Arn

SubscriptionDLQ:
  Type: AWS::SQS::Queue
  Properties:
    QueueName: my-subscription-dlq
    MessageRetentionPeriod: 1209600   # 14 days

SubscriptionDLQPolicy:
  Type: AWS::SQS::QueuePolicy
  Properties:
    Queues:
      - !Ref SubscriptionDLQ
    PolicyDocument:
      Statement:
        - Effect: Allow
          Principal:
            Service: sns.amazonaws.com
          Action: sqs:SendMessage
          Resource: !GetAtt SubscriptionDLQ.Arn
          Condition:
            ArnEquals:
              aws:SourceArn: !Ref MyTopic
```

> The subscription DLQ is separate from any DLQ configured on the downstream SQS queue. The subscription DLQ captures SNS delivery failures; the queue's own DLQ captures consumer processing failures. Both are needed.

* * *

### Configure Delivery Retry Policies for HTTP/S Endpoints

**What it is:** For HTTP/S subscriptions, SNS supports a customizable delivery retry policy that defines the number of retries, the backoff function (linear, exponential, or geometric), the minimum and maximum delay between retries, and a maximum total retry duration.

**Why it matters:** The default SNS retry policy for HTTP/S is 3 immediate retries, then 20 retries with 20-second delay, then 17 retries with linear backoff, then exponential backoff — up to 23 days total. For most production HTTP/S endpoints, this default is either too aggressive (overwhelming a recovering endpoint) or the 23-day retention is longer than your incident response window. Tune it to reflect the SLA of your endpoint.

```json
{
  "healthyRetryPolicy": {
    "numRetries": 10,
    "minDelayTarget": 20,
    "maxDelayTarget": 300,
    "numMinDelayRetries": 3,
    "numMaxDelayRetries": 5,
    "backoffFunction": "exponential"
  },
  "throttlePolicy": {
    "maxReceivesPerSecond": 10
  }
}
```

* * *

### Prefer Fan-Out via SNS → SQS → Lambda Over Direct SNS → Lambda Subscriptions

**What it is:** In the SNS fan-out pattern, each subscriber is typically an SQS queue rather than a Lambda function directly. Lambda functions then consume messages from the queue asynchronously. This introduces a durable buffering layer between message delivery and message processing.

**Why it matters:** Direct SNS → Lambda subscriptions are operationally simple and work well for many lightweight workloads, but they tightly couple message delivery to Lambda availability and concurrency behavior. SNS invokes Lambda asynchronously and retries failed deliveries automatically, but retry behavior is finite and offers less operational control than a queue-based architecture.

The SNS → SQS → Lambda pattern provides stronger durability and backpressure handling. SQS can buffer messages for up to 14 days, absorb traffic spikes, smooth bursty workloads, and isolate publishers from downstream processing delays. It also enables operational features that direct SNS → Lambda subscriptions do not provide as cleanly, including:

*   Configurable visibility timeouts
    
*   Consumer-level DLQs
    
*   Controlled concurrency scaling
    
*   Batch processing
    
*   Replay and redrive workflows
    
*   Independent retry behavior per consumer
    

This pattern is especially valuable for business-critical event processing where message durability, replayability, and operational resilience matter more than minimal infrastructure complexity.

Direct SNS → Lambda subscriptions are still appropriate for simpler workloads, lightweight notifications, or low-criticality event flows where the operational overhead of introducing SQS is not justified.

* * *

## Security

### Apply Least-Privilege IAM Policies Per Role

**What it is:** IAM policies control which principals can publish to, subscribe to, or manage SNS topics. Publishers need `sns:Publish`. Subscribers need `sns:Subscribe` and `sns:Unsubscribe`. Administrative operations (`sns:CreateTopic`, `sns:DeleteTopic`, `sns:SetTopicAttributes`) should be reserved for infrastructure automation roles, never application runtime roles.

**Why it matters:** A Lambda publisher with `sns:*` on `*` can delete your topics, change encryption settings, or add unauthorized subscriptions. Scope each role to the exact actions and topic ARNs it requires at runtime.

```yaml
PublisherRole:
  Type: AWS::IAM::Role
  Properties:
    Policies:
      - PolicyName: SNSPublisherPolicy
        PolicyDocument:
          Statement:
            - Effect: Allow
              Action:
                - sns:Publish
              Resource: !Ref MyTopic   # Specific topic ARN, not *
```

* * *

### Enable Server-Side Encryption (SSE)

**What it is:** SNS supports two SSE options: **SSE-SNS** (AWS-managed keys, no additional cost) and **SSE-KMS** (customer-managed CMK). SNS SSE encrypts messages at rest within SNS. Message delivery in transit uses TLS for supported protocols.

**Why it matters:** SNS topics often carry PII, financial events, or internal business data. SSE-SNS is free, zero-configuration, and satisfies most compliance baselines. Use SSE-KMS only when your compliance requirements demand customer-managed key control, key rotation auditability, or cross-account key sharing.

```yaml
MyTopic:
  Type: AWS::SNS::Topic
  Properties:
    TopicName: my-topic
    KmsMasterKeyId: alias/aws/sns   # SSE-SNS (free, AWS-managed)
    # For SSE-KMS: KmsMasterKeyId: !Ref MyKMSKey
```

> When using customer-managed KMS keys, ensure the SNS service principal and downstream consumers have the required KMS permissions (`kms:Decrypt`, `kms:GenerateDataKey`, etc.), or deliveries may fail.

* * *

### Restrict Topic Access with a Resource-Based Policy

**What it is:** An SNS topic policy (resource-based policy) defines which AWS accounts, IAM principals, or services are allowed to publish to or subscribe to the topic — independently of the caller's IAM identity policy. It is the primary mechanism for cross-account access and for restricting which services can publish.

**Why it matters:** Without a restrictive topic policy, any principal in your AWS account with `sns:Publish` in their IAM identity policy can publish to your topic — including services and roles that were never intended to do so. Always scope the `aws:SourceArn` condition to the specific resource (e.g., a specific API Gateway, EventBridge rule, or Lambda function) that is authorized to publish.

```yaml
MyTopicPolicy:
  Type: AWS::SNS::TopicPolicy
  Properties:
    Topics:
      - !Ref MyTopic
    PolicyDocument:
      Statement:
        - Sid: AllowPublishFromOrderService
          Effect: Allow
          Principal:
            AWS: !GetAtt OrderServiceRole.Arn
          Action: sns:Publish
          Resource: !Ref MyTopic
        - Sid: DenyPublishFromOtherAccounts
          Effect: Deny
          Principal: "*"
          Action: sns:Publish
          Resource: !Ref MyTopic
          Condition:
            StringNotEquals:
              aws:PrincipalAccount: !Ref AWS::AccountId
```

* * *

### Use VPC Endpoints for SNS

**What it is:** An SNS VPC Interface Endpoint routes traffic between your VPC resources (EC2, ECS, Lambda in VPC) and SNS privately within the AWS network, without traversing the public internet or requiring a NAT Gateway.

**Why it matters:** Without a VPC endpoint, a VPC-attached Lambda or ECS task publishing to SNS must route through a NAT Gateway, incurring data processing charges. For a high-volume publisher, this can be significant. VPC endpoints also remove the requirement to allow outbound internet access for SNS traffic, reducing attack surface and simplifying security group rules.

* * *

### Enable Data Protection Policies for PII

**What it is:** SNS Data Protection Policies allow you to define rules that audit, mask, or block messages containing sensitive data (PII, financial data, health information) based on managed or custom data identifiers. Data protection is configured at the topic level and applies to both inbound publishes and outbound deliveries.

**Why it matters:** In a fan-out topology, a single misconfigured publisher can leak PII to every subscriber — including HTTP endpoints, email addresses, and mobile push — without any subscriber being able to intercept it. Data protection policies act as a centralized control point: you can audit where PII flows, mask sensitive fields before delivery, or block publishes that contain unexpected sensitive data entirely.

```yaml
MyTopic:
  Type: AWS::SNS::Topic
  Properties:
    DataProtectionPolicy:
      Name: PIIProtectionPolicy
      Description: Audit and mask PII in transit
      Version: "2021-06-01"
      Statement:
        - Sid: AuditPII
          DataDirection: Inbound
          Principal:
            AWS: "*"
          DataIdentifier:
            - arn:aws:dataprotection::aws:data-identifier/EmailAddress
            - arn:aws:dataprotection::aws:data-identifier/CreditCardNumber
          Operation:
            Audit:
              FindingsDestination:
                CloudWatchLogs:
                  LogGroup: !Ref PIIAuditLogGroup
```

* * *

## Observability & Cost

### Monitor the Right CloudWatch Metrics

**What it is:** SNS publishes several CloudWatch metrics out of the box. The most operationally important are:

| Metric | What it tells you |
| --- | --- |
| `NumberOfNotificationsDelivered` | Successful delivery count per protocol |
| `NumberOfNotificationsFailed` | Failed deliveries after all retries — the critical one |
| `NumberOfNotificationsFilteredOut` | Messages dropped by subscription filter policies |
| `NumberOfNotificationsFilteredOut-InvalidAttributes` | Messages dropped due to malformed attributes — publisher bugs |
| `NumberOfNotificationsFilteredOut-NoMessageAttributes` | Messages dropped because no attributes were present |
| `PublishSize` | Message size distribution — alerts to unexpectedly large payloads |
| `SMSSuccessRate` | Delivery success rate for SMS subscriptions |

**Why it matters:** `NumberOfNotificationsFailed` is the single most important metric — it represents events permanently lost after SNS exhausted all retry attempts. This metric has no alarm by default. A spike in `NumberOfNotificationsFilteredOut-InvalidAttributes` indicates a publisher bug where message attributes are malformed, silently dropping messages for all subscribers with filter policies.

* * *

### Enable Delivery Status Logging

**What it is:** SNS delivery status logging writes a CloudWatch Logs entry for every delivery attempt — successful or failed — for SQS, Lambda, HTTP/S, and other subscription protocols. It is configured per protocol at the topic level by specifying a success and failure sample rate and a CloudWatch Logs IAM role.

**Why it matters:** Without delivery status logging, you know SNS delivered N messages (from the metric) but you cannot determine *which* messages failed, *why* they failed, or *when* the failure occurred. Delivery status logs give you per-message visibility: the `status`, `statusCode`, `responseBody`, `dwellTimeMs`, and `destination` for every attempt. This is essential for diagnosing intermittent HTTP endpoint failures or Lambda throttling events.

> Set the success sample rate to 5–10% and the failure sample rate to 100%. You want full visibility on failures without paying for logging every successful delivery at high throughput.

* * *

### Set Alarms on Delivery Failures and Filtered-Out Messages

**What it is:** CloudWatch Alarms can trigger notifications (via another SNS topic, PagerDuty, Slack via EventBridge) when SNS delivery metrics breach thresholds. At minimum, alarm on `NumberOfNotificationsFailed` greater than 0 per protocol.

**Why it matters:** A delivery failure is a lost event. Unlike a queue with a DLQ, SNS delivery failures (for subscribers without a subscription DLQ) are permanent and unrecoverable. A zero-tolerance alarm fires the moment the first event is lost, rather than when a silent backlog has accumulated.

```yaml
SNSDeliveryFailureAlarm:
  Type: AWS::CloudWatch::Alarm
  Properties:
    AlarmName: !Sub "${MyTopic}-SQS-DeliveryFailures"
    MetricName: NumberOfNotificationsFailed
    Namespace: AWS/SNS
    Dimensions:
      - Name: TopicName
        Value: !GetAtt MyTopic.TopicName
    Statistic: Sum
    Period: 60
    EvaluationPeriods: 1
    Threshold: 0
    ComparisonOperator: GreaterThanThreshold
    TreatMissingData: notBreaching
    AlarmActions:
      - !Ref OpsAlertTopic

FilteredOutInvalidAttributesAlarm:
  Type: AWS::CloudWatch::Alarm
  Properties:
    AlarmName: !Sub "${MyTopic}-InvalidAttributesDropped"
    MetricName: NumberOfNotificationsFilteredOut-InvalidAttributes
    Namespace: AWS/SNS
    Dimensions:
      - Name: TopicName
        Value: !GetAtt MyTopic.TopicName
    Statistic: Sum
    Period: 300
    EvaluationPeriods: 1
    Threshold: 0
    ComparisonOperator: GreaterThanThreshold
    TreatMissingData: notBreaching
    AlarmActions:
      - !Ref OpsAlertTopic
```

* * *

### Tag Topics for Cost Allocation

**What it is:** AWS resource tags applied to SNS topics can be activated as Cost Allocation Tags in the Billing console, enabling AWS Cost Explorer to break down SNS spending by team, service, or environment.

**Why it matters:** In a shared account with dozens of topics, SNS costs appear as a single line item without tagging. High-volume topics — especially those driving SMS notifications or large-payload deliveries — can drive surprising costs. Tags make per-team or per-service SNS cost accountability possible and support chargeback models in platform teams.

```yaml
MyTopic:
  Type: AWS::SNS::Topic
  Properties:
    Tags:
      - Key: Team
        Value: order-service
      - Key: Environment
        Value: production
      - Key: CostCenter
        Value: platform-engineering
```

* * *

## Operations

### Define Topics and Subscriptions with Infrastructure as Code

**What it is:** SNS topics, subscriptions, topic policies, subscription filter policies, DLQ configurations, and alarms should all be defined in CloudFormation, SAM, CDK, or Terraform — not created manually through the console. Topic ARNs should be passed to publishing services as environment variables, never hardcoded.

**Why it matters:** Manually created topics accumulate configuration drift, have no change history, and cannot be reliably recreated after an accidental deletion. IaC also makes it trivial to enforce consistent settings (SSE, subscription DLQ, delivery logging, alarms) across all topics by using shared templates or constructs. A hardcoded topic ARN in a Lambda environment variable is a silent misconfiguration waiting to surface in a disaster recovery scenario.

* * *

### Have a Cross-Account Subscription Strategy

**What it is:** SNS supports cross-account subscriptions, where a topic in Account A delivers to an SQS queue or Lambda in Account B. This requires both a topic policy in Account A (allowing the other account to subscribe) and a queue/Lambda resource policy in Account B (allowing SNS to deliver from Account A's topic).

**Why it matters:** Cross-account fan-out is a common pattern in platform/product account models, where a platform team owns the event bus (SNS topic) and product teams consume from it in their own accounts. Without correctly scoped policies on both sides, delivery fails silently — SNS reports the subscription as active, but delivery attempts fail with authorization errors that only appear in delivery status logs. Always lock down `aws:SourceAccount` and `aws:SourceArn` conditions on both sides.

```yaml
# In the subscriber account — SQS queue policy allowing SNS from Account A
QueuePolicy:
  Type: AWS::SQS::QueuePolicy
  Properties:
    Queues:
      - !Ref SubscriberQueue
    PolicyDocument:
      Statement:
        - Effect: Allow
          Principal:
            Service: sns.amazonaws.com
          Action: sqs:SendMessage
          Resource: !GetAtt SubscriberQueue.Arn
          Condition:
            ArnEquals:
              aws:SourceArn: !Sub "arn:aws:sns:us-east-1:PUBLISHER_ACCOUNT_ID:my-topic"
```

* * *

### Verify Subscription Confirmation for HTTP/S Endpoints

**What it is:** When an HTTP/S endpoint is subscribed to an SNS topic, SNS sends a `SubscriptionConfirmation` request to the endpoint URL containing a confirmation token. The endpoint must call `ConfirmSubscription` with that token within 3 days or the subscription is automatically deleted.

**Why it matters:** An HTTP/S subscription that is never confirmed is silently inactive — SNS will not deliver messages to it. In automated deployments, the confirmation step is easy to miss: the CloudFormation stack creates the subscription successfully (from the IaC perspective), but the endpoint never confirms, and SNS silently removes the subscription. Implement confirmation handling in every HTTP/S subscriber and add a `SubscriptionConfirmed` log entry at startup to detect missing confirmations early.

* * *

## Quick Reference

> **Must** = required for any production topic regardless of workload. **Optional** = strongly advisable but context-dependent.

| # | Practice | Priority | When to apply |
| --- | --- | --- | --- |
| **Topic Design** |  |  |  |
| 1 | Choose Standard vs FIFO intentionally | Must | Always — never default without considering ordering, throughput, and subscriber types |
| 2 | Define topic granularity (per domain/aggregate, not God topic) | Must | All production architectures |
| 3 | Plan for 256 KB limit — use claim-check for large payloads | Must | Any topic where payload size is variable |
| **Message Design** |  |  |  |
| 4 | Use a consistent event envelope | Optional | Always — across all producers |
| 5 | Include correlation IDs in every message | Must | Always |
| 6 | Use message attributes for routing/filtering metadata | Must | Any topic with multiple subscribers that need different subsets |
| 7 | Design for schema evolution | Must | All topics expected to outlive a single sprint |
| **Publishers** |  |  |  |
| 8 | Use PublishBatch for high-volume publishers | Must | Always for high-volume producers |
| 9 | Handle partial failures in batch publishes | Must | Any code that calls PublishBatch |
| 10 | Use MessageGroupId and explicit MessageDeduplicationId for FIFO | Must | All FIFO topic publishers |
| **Subscriptions** |  |  |  |
| 11 | Use subscription filter policies | Must | Any subscriber that cares about fewer than all events on the topic |
| 12 | Enable raw message delivery for SQS subscribers | Must | All SQS and Lambda subscriptions |
| 13 | Configure a DLQ for every subscription | Must | All subscriptions |
| 14 | Configure delivery retry policy for HTTP/S endpoints | Must | All HTTP/S subscriptions |
| 15 | Prefer SNS → SQS → Lambda over direct SNS → Lambda | Must | Any subscriber where durability matters |
| **Security** |  |  |  |
| 16 | Apply least-privilege IAM per publisher/admin role | Must | Always |
| 17 | Enable Server-Side Encryption (SSE-SNS or SSE-KMS) | Must | Always — SSE-SNS is free |
| 18 | Restrict topic with resource-based policy | Must | Any topic in a multi-team or multi-account environment |
| 19 | Use VPC endpoints for SNS | Optional | Any VPC-attached publisher calling SNS at high volume |
| 20 | Enable Data Protection Policies for PII topics | Must | Any topic that may carry PII, PHI, or financial data |
| **Observability & Cost** |  |  |  |
| 21 | Monitor NumberOfNotificationsFailed per protocol | Must | Always — at minimum this one metric |
| 22 | Enable delivery status logging (100% failure, 5–10% success) | Optional | All production topics |
| 23 | Set alarms on delivery failures and invalid attribute drops | Must | All production topics — alarm on failures > 0 |
| 24 | Tag topics with cost allocation tags | Must | Any shared or multi-team AWS account |
| **Operations** |  |  |  |
| 25 | Define topics and subscriptions in IaC | Must | Always |
| 26 | Document cross-account subscription policy on both sides | Optional | Any cross-account fan-out topology |
| 27 | Implement and verify HTTP/S subscription confirmation | Must | All HTTP/S subscriptions |

* * *

The current checklist is a starting point. Adapt it to your team's context, add items that reflect your specific compliance requirements or architectural patterns, and remove items that genuinely do not apply to your workload. The goal is not a perfect score — it is a shared, team-maintained definition of what a production-ready SNS topic looks like for your organization. Thanks, and happy coding.