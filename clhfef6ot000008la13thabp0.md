---
title: "Consuming AWS SQS messages in order"
datePublished: Mon May 08 2023 22:12:41 GMT+0000 (Coordinated Universal Time)
cuid: clhfef6ot000008la13thabp0
slug: consuming-aws-sqs-messages-in-order
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1683333070085/7912715d-252a-4523-a244-90ffb53c3ec3.png
tags: csharp, aws, net, aws-sqs

---

Event-driven architectures present challenges at every turn. One such challenge is ensuring the consumption of messages in order. To address this challenge, AWS offers [SQS FIFO queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/FIFO-queues.html):

> *FIFO (First-In-First-Out)* queues have all the capabilities of the standard queues but are designed to enhance messaging between applications when the order of operations and events is critical, or where duplicates can't be tolerated.

There are two additional attributes required to guarantee [message ordering](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/FIFO-queues-message-order.html) and [exactly-once processing](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/FIFO-queues-exactly-once-processing.html) from the producer's perspective:

* `MessageGroupID`: Messages within a single group are always processed one at a time in the order in which they were received by the queue.
    
* `MessageDeduplicationID`: Any messages sent with the same MessageDuplicationID are accepted successfully, but only one is delivered within the 5-minute deduplication period. We can enable the content-based deduplication at the queue level to make this attribute optional.
    

From the consumer's perspective, there is nothing needs to be changed. Three rules govern the order in which messages leave the queue:

1. Return the oldest message where no other message with the same `MessageGroupId` is in flight.
    
2. Return as many messages with the same `MessageGroupId` as possible.
    
3. If a message batch is still not full, go back to the first rule. As a result, a single batch can contain messages from multiple `MessageGroupIds`.
    

With these simple rules, the queue ensures messages from the same message group are not served to more than one consumer simultaneously. However, not everything is perfect throughput goes down from nearly unlimited to 3,000 messages per second with batching (300 messages per second without batching) against the API. If we require higher throughput, we can enable the [high throughput mode](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/high-throughput-fifo.html#partitions-and-data-distribution), which will support up to 30,000 messages per second with batching (3,000 messages per second without batching). Let's create a producer and consumer application to demonstrate how simple it is to use an AWS SQS FIFO queue. First, we need to complete the following prerequisites:

* An [**IAM User**](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html#id_users_create_console) with programmatic access.
    
* Install [**AWS CLI**](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-creds).
    
* Install [**AWS SAM CLI**](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html).
    

To create the queue, we will be using AWS SAM (it can also be done manually through the AWS Console). Create a `template.yaml` file with the following content:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: >
  SQS
Resources:
  SQSQueue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: "mySQSQueue.fifo"
      FifoQueue: true
      ContentBasedDeduplication: true
      ReceiveMessageWaitTimeSeconds: 5
Outputs:
  SQSQueueUrl:
    Description: SQS Url
    Value: !Ref SQSQueue
```

Run `sam build` and `sam deploy` to provision the resource. Execute the following commands to create the projects and solution:

```bash
dotnet new console -n Consumer -o ./src/Consumer
dotnet new console -n Producer -o ./src/Producer
dotnet new sln -n SQSSandbox
dotnet sln add --in-root src/Consumer
dotnet sln add --in-root src/Producer
dotnet add src/Producer package AWSSDK.SQS
dotnet add src/Producer package AWSSDK.Extensions.NETCore.Setup
dotnet add src/Producer package Microsoft.Extensions.Configuration.Json
dotnet add src/Producer package Microsoft.Extensions.DependencyInjection
dotnet add src/Consumer package AWSSDK.SQS
dotnet add src/Consumer package AWSSDK.Extensions.NETCore.Setup
dotnet add src/Consumer package Microsoft.Extensions.Configuration.Json
dotnet add src/Consumer package Microsoft.Extensions.DependencyInjection
```

Open the `Producer` project and update the `Program.cs` file as follows:

```csharp
using Amazon.SQS;
using Amazon.SQS.Model;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;

var configurationBuilder = new ConfigurationBuilder()
    .AddJsonFile("appsettings.json");

var options = configurationBuilder.Build().GetAWSOptions();
var services = new ServiceCollection()
    .AddDefaultAWSOptions(options)
    .AddAWSService<IAmazonSQS>();

var provider = services.BuildServiceProvider();
var url = "<MY_QUEUE_URL>";
var sqsClient = provider.GetService<IAmazonSQS>()!;
for (int i = 0; i < 100; i++)
{
    var messageGroupId = Random.Shared.Next(0, 5).ToString();
    var body = $"@@{messageGroupId}@@{Guid.NewGuid()}";
    var request = new SendMessageRequest(url, body)
    {
        MessageGroupId = messageGroupId
    };
    await sqsClient.SendMessageAsync(request);
    Console.WriteLine($"{body} sent");
}
```

Add an `appsettings.json` with the following content:

```json
{
	"AWS": {
		"Profile": "default",
		"Region": "<MY_REGION>"
	}
}
```

Open the `Consumer` project and update the `Program.cs` file as follows:

```csharp
using Amazon.SQS;
using Amazon.SQS.Model;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;

var configurationBuilder = new ConfigurationBuilder()
    .AddJsonFile("appsettings.json");

var options = configurationBuilder.Build().GetAWSOptions();
var services = new ServiceCollection()
    .AddDefaultAWSOptions(options)
    .AddAWSService<IAmazonSQS>();

var provider = services.BuildServiceProvider();
var url = "<MY_QUEUE_URL>";
var sqsClient = provider.GetService<IAmazonSQS>()!;
while (true)
{
    var receiveRequest = new ReceiveMessageRequest
    {
        QueueUrl = url,
        MaxNumberOfMessages = 10, 
        WaitTimeSeconds = 5
    };

    var result = await sqsClient.ReceiveMessageAsync(receiveRequest);
    if (result.Messages.Any())
    {
        var batchId = Guid.NewGuid().ToString();
        var total = result.Messages.Count;
        var current = 1;
        var batch = new List<DeleteMessageBatchRequestEntry>();
        foreach (var message in result.Messages)
        {
            Console.WriteLine($"Batch({batchId}) {current} of {total} with {message.Body} received");
            current++;
            batch.Add(new DeleteMessageBatchRequestEntry() { ReceiptHandle = message.ReceiptHandle, Id=message.MessageId });
            await Task.Delay(Random.Shared.Next(1000, 2500));
        }
        await sqsClient.DeleteMessageBatchAsync(url, batch);
    }
    else
    {
        Console.WriteLine("No messages available");
        await Task.Delay(TimeSpan.FromSeconds(5));
    }
}
```

Add an `appsettings.json` as we did with the consumer. Run `dotnet run --project src/Consumer` in two or more console windows, and `dotnet run --project src/Producer` to see everything in action. You can see the code and scripts [here](https://github.com/raulnq/aws-sqs-lambda). Thanks, and happy coding.