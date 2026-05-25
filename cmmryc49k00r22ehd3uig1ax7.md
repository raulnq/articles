---
title: "Moto Server: Our Own Local AWS"
datePublished: 2026-03-15T16:11:20.602Z
cuid: cmmryc49k00r22ehd3uig1ax7
slug: moto-server-our-own-local-aws
cover: https://cdn.hashnode.com/uploads/covers/625b81432f84e0d4fc33dca5/033b8e60-c8b4-4b71-993b-19b92203b2ce.png
tags: aws, dotnet, sqs, sns, moto-server

---

[Moto](https://github.com/getmoto/moto) is an open-source Python library originally designed for mocking AWS services. Moto Server is its standalone HTTP server mode, which exposes a fully functional fake AWS API that any client — regardless of programming language — can talk to using the standard AWS SDK. At its core, Moto Server intercepts requests that would normally go to `https://<service>.amazonaws.com` and handles them locally, maintaining in-memory state that faithfully mimics real AWS behavior. Moto currently supports over 80 AWS services, including:

*   **SNS** (Simple Notification Service)
    
*   **SQS** (Simple Queue Service)
    
*   **S3** (Simple Storage Service)
    
*   **DynamoDB**
    
*   **Lambda**
    
*   **IAM**, **EC2**, **Kinesis**, **Secrets Manager**, and many more
    

The key insight is that Moto Server speaks the exact same HTTP protocol as real AWS. Our application code — including the AWS SDK — does not need any modification to switch between Moto Server and real AWS. **The only change is the endpoint URL**. Moto Server does not require real credentials, and it does not validate them. However, AWS SDKs still expect credentials to exist, so we usually must provide dummy credentials in environment variables or config files.

## Why Use Moto Server?

### The Problem with Testing Against Real AWS

Testing cloud-native applications presents a persistent challenge: real AWS services are external, stateful, and cost money. Running integration tests against live infrastructure leads to:

*   **Slow feedback loops**: Network latency adds seconds to every test case.
    
*   **Flaky behavior**: Network timeouts, throttling, and eventual consistency make tests non-deterministic.
    
*   **Cloud cost**: High-frequency test pipelines can generate unexpected AWS bills.
    
*   **Environment coupling**: Developers share the same dev/staging account, causing test data collisions.
    
*   **No offline development**: We cannot work on a plane or in an environment with no internet.
    

### How Moto Server Solves These Problems

*   **Slow feedback loops**: Runs locally; no network round-trips to AWS.
    
*   **Flaky behavior**: Deterministic in-memory state; no eventual consistency.
    
*   **Cloud cost**: Completely free; runs as a local Docker container.
    
*   **Environment coupling**: Each developer (or CI run) spins up their own isolated instance.
    
*   **No offline development**: Runs entirely offline.
    

### Moto Server vs. LocalStack

A natural question is: **why not use LocalStack?** Both tools simulate AWS locally, but they differ in philosophy and scope.

*   **LocalStack** is a commercial product (with a free tier) that runs each AWS service in a dedicated process and focuses on maximum fidelity, including networking emulation.
    
*   **Moto Server** is a pure open-source, community-driven project. It runs all services in a single process and is particularly strong for SNS, SQS, S3, and DynamoDB.
    

For most integration testing scenarios — especially message-broker flows with SNS and SQS — Moto Server is lightweight, zero-configuration, and entirely sufficient.

## Running Moto Server Locally with Docker

Moto Server ships as an official Docker image. Spinning it up takes one command.

```shell
docker run --rm -d --name moto-server -p 5000:5000 motoserver/moto:latest
```

What each flag does:

*   `--rm`: Removes the container when it stops, keeping our environment clean.
    
*   `-d`: Runs in detached (background) mode.
    
*   `--name moto-server`: Gives the container a predictable name for referencing later.
    
*   `-p 5000:5000`: Maps port 5000 on our host machine to port 5000 inside the container. Moto Server listens on port 5000 by default.
    

Once started, the server is immediately ready at `http://localhost:5000`. To verify the server is running navigate to `http://localhost:5000/moto-api/`, we should see a dashboard. Moto Server also exposes a reset endpoint that is useful between test runs `http://localhost:5000/moto-api/reset` (POST).

## Using Moto Server

Let's create a couple of applications—a message producer and its consumer—to see Moto Server in action.

### Setting Up AWS Resources with the AWS CLI

Before our applications can publish or consume messages, the AWS resources (SNS topic, SQS queue, and the subscription linking them) must exist. In a real project, we would employ Infrastructure as Code tools like Terraform, CDK, or CloudFormation. However, for local development with Moto Server, the AWS CLI offers the quickest setup, though it can also be integrated with these tools.

No special profile configuration is needed. Moto Server does not validate credentials — it accepts any value the CLI sends. The only flag that matters is `--endpoint-url`, which redirects the request from the real AWS endpoint to our local Moto instance. As long as the CLI has some credentials available (from environment variables, our default profile, or any other source), the request will be signed and Moto will accept it. All commands in this section use only `--endpoint-url http://localhost:5000`.

**Creating the SNS Topic**

```shell
aws sns create-topic --name my-topic --endpoint-url http://localhost:5000
```

**Creating the SQS Queue**

```shell
aws sqs create-queue --queue-name my-queue --endpoint-url http://localhost:5000
```

**Subscribing the SQS Queue to the SNS Topic**

The SNS subscription requires the SQS queue's ARN, not its URL. Retrieve it with:

```shell
aws sqs get-queue-attributes --queue-url <QUEUE_URL> --attribute-names QueueArn --endpoint-url http://localhost:5000
```

This is the step that wires the two services together. When a message is published to the SNS topic, SNS will automatically deliver it to all subscribers — in this case, the SQS queue.

```shell
aws sns subscribe --topic-arn <TOPIC_ARN> --protocol sqs --notification-endpoint <QUEUE_ARN> --endpoint-url http://localhost:5000
```

### Building the SNS Producer

The producer application is a minimal .NET Web API that accepts an HTTP request and publishes an event to the SNS topic.

```shell
dotnet new webapi -n SnsProducer
cd SnsProducer
dotnet add package AWSSDK.SimpleNotificationService
dotnet add package AWSSDK.Extensions.NETCore.Setup
```

Why these packages:

*   `AWSSDK.SimpleNotificationService`: The AWS SDK for the SNS service.
    
*   `AWSSDK.Extensions.NETCore.Setup`: Provides `GetAWSOptions()` and `AddAWSService<T>()`, the idiomatic integration between the AWS SDK and .NET's dependency injection container.
    

The SDK's `GetAWSOptions()` extension reads from the reserved `AWS` section of `appsettings.json`. This is the only place that needs to change between environments — the application code is identical across all of them.

```json
{
  "AWS": {
    "Region": "us-east-1",
    "ServiceURL": "http://localhost:5000",
  }
}
```

Why two keys in the `AWS` section?

*   `Region`: the logical AWS region. The SDK uses this to construct ARNs and to select the correct real AWS endpoint under normal circumstances.
    
*   `ServiceURL`: overrides the endpoint entirely, redirecting all requests to Moto Server instead of real AWS. When this is set, `Region` no longer drives endpoint selection — it is used purely for ARN construction.
    

In rare cases we could need to setup the `AuthenticationRegion` key. This key tells the SDK which region to embed in the [AWS Signature Version 4](https://docs.aws.amazon.com/general/latest/gr/signature-version-4.html) request signature. Both `Region` and `AuthenticationRegion` must match the region used when creating our AWS resources. Open the `Program.cs` file and replace the content with:

```csharp
// Program.cs
using System.Text.Json;
using Amazon.SimpleNotificationService;
using Amazon.SimpleNotificationService.Model;

var builder = WebApplication.CreateBuilder(args);

// GetAWSOptions() reads the "AWS" section from appsettings.json.
// Credentials are resolved from environment variables by the SDK automatically.
builder.Services.AddDefaultAWSOptions(builder.Configuration.GetAWSOptions());

// Registers a fully configured, singleton IAmazonSimpleNotificationService.
builder.Services.AddAWSService<IAmazonSimpleNotificationService>();

var app = builder.Build();

const string TopicArn = "<TOPIC_ARN>";

var jsonOptions = new JsonSerializerOptions
{
    PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
    WriteIndented = false
};

app.MapPost("/api/events", async (
    IAmazonSimpleNotificationService sns,
    ILogger<Program> logger,
    CancellationToken cancellationToken) =>
{
    var eventId = Guid.NewGuid();

    var orderEvent = new MyEvent(
        EventId: eventId,
        Message: $"New event {eventId}",
        CreatedAt: DateTimeOffset.UtcNow
    );

    // Serialize the event to JSON. The message body is always a string in SNS.
    var messageBody = JsonSerializer.Serialize(orderEvent, jsonOptions);

    var publishRequest = new PublishRequest
    {
        TopicArn = TopicArn,
        Message = messageBody,
        // MessageGroupId and MessageDeduplicationId are required for FIFO topics.
        // For standard topics, Message Attributes are useful for filtering.
        MessageAttributes = new Dictionary<string, MessageAttributeValue>
        {
            ["EventType"] = new MessageAttributeValue
            {
                DataType = "String",
                StringValue = nameof(MyEvent)
            },
            ["Version"] = new MessageAttributeValue
            {
                DataType = "String",
                StringValue = "1.0"
            }
        }
    };

    logger.LogInformation(
        "Publishing {EventType} for order {EventId} to topic {TopicArn}",
        nameof(MyEvent), eventId, TopicArn);

    try
    {
        var response = await sns.PublishAsync(publishRequest, cancellationToken);

        logger.LogInformation(
            "Published {EventType} successfully. MessageId: {MessageId}",
            nameof(MyEvent), response.MessageId);
    }
    catch (Exception ex)
    {
        logger.LogError(ex,
            "Failed to publish {EventType} for order {EventId}",
            nameof(MyEvent), eventId);
        throw;
    }

    return Results.Ok();
});

app.Run();

record MyEvent(
    Guid EventId,
    string Message,
    DateTimeOffset CreatedAt
);
```

### Building the SQS Consumer

The consumer is a .NET Worker Service — a long-running background process that continuously polls the SQS queue and processes incoming messages.

```csharp
dotnet new worker -n SqsConsumer
cd SqsConsumer
dotnet add package AWSSDK.SQS
dotnet add package AWSSDK.Extensions.NETCore.Setup
```

Just like the producer, the entire consumer lives in a single file. Open the `Program.cs` file and replace the content with:

```csharp
// Program.cs
using System.Text.Json;
using System.Text.Json.Serialization;
using Amazon.SQS;
using Amazon.SQS.Model;

var builder = Host.CreateApplicationBuilder(args);

// GetAWSOptions() reads the "AWS" section from appsettings.json.
// Credentials are resolved from environment variables by the SDK automatically.
builder.Services.AddDefaultAWSOptions(builder.Configuration.GetAWSOptions());

// Registers a fully configured, singleton IAmazonSQS.
builder.Services.AddAWSService<IAmazonSQS>();

builder.Services.AddHostedService<SqsPollingWorker>();

var host = builder.Build();
host.Run();

public class SqsPollingWorker(IAmazonSQS sqs, ILogger<SqsPollingWorker> logger)
    : BackgroundService
{
    // QueueUrl is hardcoded here for simplicity in this example.
    // In a production codebase, read it from configuration or AWS Parameter Store.
    private const string QueueUrl =
        "<QUEUE_URL>";

    // Long polling: holds the connection open for up to 20 seconds.
    // This drastically reduces empty responses and cuts API call costs.
    private const int WaitTimeSeconds = 20;

    // Retrieve up to 10 messages per poll request (the SQS maximum).
    private const int MaxNumberOfMessages = 10;

    // VisibilityTimeout hides the message from other consumers while processing.
    // If processing fails and the message is not deleted, it reappears after this timeout.
    // Set it comfortably above the maximum expected processing time.
    private const int VisibilityTimeoutSeconds = 30;

    private static readonly JsonSerializerOptions JsonOptions = new()
    {
        PropertyNameCaseInsensitive = true
    };

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        logger.LogInformation("SQS consumer started. Polling queue: {QueueUrl}", QueueUrl);

        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await PollAndProcessAsync(stoppingToken);
            }
            catch (OperationCanceledException)
            {
                // Expected when the host is shutting down. Break cleanly.
                break;
            }
            catch (Exception ex)
            {
                // Log unexpected errors but keep the loop running.
                // Without this, a single transient error would kill the worker.
                logger.LogError(ex, "Unexpected error in SQS polling loop. Retrying in 5 seconds.");
                await Task.Delay(TimeSpan.FromSeconds(5), stoppingToken);
            }
        }

        logger.LogInformation("SQS consumer stopped.");
    }

    private async Task PollAndProcessAsync(CancellationToken cancellationToken)
    {
        var receiveRequest = new ReceiveMessageRequest
        {
            QueueUrl              = QueueUrl,
            WaitTimeSeconds       = WaitTimeSeconds,
            MaxNumberOfMessages   = MaxNumberOfMessages,
            VisibilityTimeout     = VisibilityTimeoutSeconds,
            MessageSystemAttributeNames        = ["All"],
            MessageAttributeNames = ["All"]
        };

        var response = await sqs.ReceiveMessageAsync(receiveRequest, cancellationToken);

        if (response.Messages == null)
            return; // No messages — long poll timed out. Loop again immediately.

        if (response.Messages.Count == 0)
            return; // No messages — long poll timed out. Loop again immediately.

        logger.LogInformation("Received {Count} message(s) from SQS.", response.Messages.Count);

        // Process each message independently so a failure on one does not
        // prevent the others from being handled.
        await Task.WhenAll(response.Messages
            .Select(msg => ProcessMessageAsync(msg, cancellationToken)));
    }

    private async Task ProcessMessageAsync(Message sqsMessage, CancellationToken cancellationToken)
    {
        try
        {
            // Step 1: Unwrap the SNS envelope.
            // When SNS delivers to SQS it wraps the payload in a notification envelope.
            // The actual event lives inside envelope.Message as a JSON string.
            var envelope = JsonSerializer.Deserialize<SnsEnvelope>(sqsMessage.Body, JsonOptions);

            if (envelope is null || envelope.Type != "Notification")
            {
                logger.LogWarning(
                    "Received non-notification message type '{Type}'. Deleting.",
                    envelope?.Type ?? "unknown");
                // Delete malformed or unexpected messages to avoid infinite retries.
                await DeleteMessageAsync(sqsMessage, cancellationToken);
                return;
            }

            var eventType = envelope.MessageAttributes?.GetValueOrDefault("EventType")?.Value;

            logger.LogDebug(
                "Processing SNS notification. MessageId={MessageId}, EventType={EventType}",
                envelope.MessageId, eventType ?? "unknown");

            // Step 2: Deserialize the inner payload.
            var myEvent = JsonSerializer.Deserialize<MyEvent>(envelope.Message, JsonOptions);

            if (myEvent is null)
            {
                logger.LogError(
                    "Failed to deserialize MyEvent from message {MessageId}. Body: {Body}",
                    sqsMessage.MessageId, envelope.Message);
                // Do NOT delete — let the visibility timeout expire so the message
                // can be retried or moved to a dead-letter queue.
                return;
            }

            // Step 3: Process the event.
            logger.LogInformation(
                "Processing MyEvent: EventId={EventId}, Message={Message}",
                myEvent.EventId, myEvent.Message);

            // Simulate async processing work (e.g., writing to a database,
            // calling a downstream service, triggering a workflow).
            await Task.Delay(TimeSpan.FromMilliseconds(200), cancellationToken);

            logger.LogInformation(
                "Successfully processed MyEvent for EventId={EventId}", myEvent.EventId);

            // Step 4: Delete the message ONLY after successful processing.
            // This is the at-least-once delivery guarantee — if we crash before
            // deleting, SQS will redeliver the message.
            await DeleteMessageAsync(sqsMessage, cancellationToken);
        }
        catch (JsonException ex)
        {
            logger.LogError(ex,
                "JSON deserialization failed for message {MessageId}. Deleting to avoid poison-pill loop.",
                sqsMessage.MessageId);
            await DeleteMessageAsync(sqsMessage, cancellationToken);
        }
        catch (Exception ex)
        {
            logger.LogError(ex,
                "Failed to process message {MessageId}. It will reappear after the visibility timeout.",
                sqsMessage.MessageId);
            // Do NOT delete — let SQS retry (and eventually the DLQ) handle it.
        }
    }

    private async Task DeleteMessageAsync(Message sqsMessage, CancellationToken cancellationToken)
    {
        try
        {
            await sqs.DeleteMessageAsync(QueueUrl, sqsMessage.ReceiptHandle, cancellationToken);
            logger.LogDebug("Deleted message {MessageId} from queue.", sqsMessage.MessageId);
        }
        catch (Exception ex)
        {
            // Deletion failure is non-fatal — the message will reappear after the
            // visibility timeout and may be processed again (idempotency matters here).
            logger.LogWarning(ex,
                "Failed to delete message {MessageId}. It may be processed again.",
                sqsMessage.MessageId);
        }
    }
}

// Represents the SNS notification envelope that wraps every message
// delivered to SQS via an SNS subscription. The actual event payload
// lives inside the Message field as a JSON-encoded string.
record SnsEnvelope(
    [property: JsonPropertyName("Type")]      string Type,
    [property: JsonPropertyName("MessageId")] string MessageId,
    [property: JsonPropertyName("TopicArn")]  string TopicArn,
    [property: JsonPropertyName("Message")]   string Message,
    [property: JsonPropertyName("Timestamp")] string Timestamp,
    [property: JsonPropertyName("MessageAttributes")]
        Dictionary<string, SnsMessageAttribute>? MessageAttributes
);

record SnsMessageAttribute(
    [property: JsonPropertyName("Type")]  string Type,
    [property: JsonPropertyName("Value")] string Value
);

// Must match the shape serialized by the producer.
record MyEvent(
    Guid EventId,
    string Message,
    DateTimeOffset CreatedAt
);
```

### Full End-to-End Flow

Go to the SnsProducer folder and run the command `dotnet run`, then go to the SqsConsumer folder and run the command `dotnet run` and trigger and event by sending a request to `http://localhost:5050/api/events` (POST).

You can find all the code [here](https://github.com/raulnq/moto-server). Thanks, and happy coding