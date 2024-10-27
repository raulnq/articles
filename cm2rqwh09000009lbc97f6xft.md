---
title: "Getting Started with Amazon EventBridge for .NET Developers"
datePublished: Sun Oct 27 2024 15:29:06 GMT+0000 (Coordinated Universal Time)
cuid: cm2rqwh09000009lbc97f6xft
slug: getting-started-with-amazon-eventbridge-for-net-developers
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1729886784317/fecbba9d-7a25-4a9f-94b6-1c858af223ab.png
tags: aws, dotnet, aws-lambda, aws-eventbridge

---

To develop event-driven applications in AWS, we often use services like [SQS](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html) to send commands and [SNS](https://docs.aws.amazon.com/sns/latest/dg/welcome.html) to publish events. As an alternative to SNS, AWS also offers [Amazon EventBridge](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-what-is.html).

> EventBridge is a serverless service that uses events to connect application components together, making it easier for you to build scalable event-driven applications.

Amazon EventBridge is designed with features that extend beyond SNS, such as advanced **event** **filtering**, **transformation**, and **routing**. It handles structured events that can follow a specific schema and provides more integration options than SNS. These new features come at a slightly higher cost than SNS, and currently, it does not include a feature to ensure event ordering, which SNS already supports.

The key components to understand about Amazon EventBridge are:

* **Events**: An event is a simple JSON object.
    
* **Sources**: An event comes from AWS services, custom applications, and partners (SaaS applications).
    
* **Event Bus**: An event bus acts as a router that receives events and delivers them to zero or more targets.
    
* **Rules**: An event bus can have multiple rules. A rule filters and routes events to multiple targets when they match a [pattern](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-event-patterns.html).
    
* **Targets**: A target is the destination where events are sent.
    

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1729897675497/172e194e-851f-464e-b6cc-68cd6980e6ab.png align="center")

Let's see how this works in practice by creating two Lambda functions, one to produce events and the other to consume them.

## Pre-requisites

* Have an [**IAM User**](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html#id_users_create_console) with programmatic access.
    
* Install the Amazon Lambda Templates (`dotnet new -i Amazon.Lambda.Templates`)
    
* Install the Amazon Lambda Tools (`dotnet tool install -g Amazon.Lambda.Tools`)
    
* Install [**AWS SAM CLI**](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html).
    

## The Backend Services

Run the following commands to set up our Lambda functions:

```powershell
dotnet new lambda.EmptyFunction -n MyLambda -o .
dotnet add src/MyLambda package AWSSDK.EventBridge
dotnet add src/MyLambda package Amazon.Lambda.APIGatewayEvents
dotnet add src/MyLambda package Amazon.Lambda.CloudWatchEvents
dotnet new sln -n EventBridge
dotnet sln add --in-root src/MyLambda
```

Open the `Program.cs` file and update the content as follows:

```csharp
using Amazon.EventBridge;
using Amazon.EventBridge.Model;
using Amazon.Lambda.APIGatewayEvents;
using Amazon.Lambda.CloudWatchEvents;
using Amazon.Lambda.Core;
using System.Text.Json;

[assembly: LambdaSerializer(typeof(Amazon.Lambda.Serialization.SystemTextJson.DefaultLambdaJsonSerializer))]
namespace MyLambda;

public class Function
{
    private readonly AmazonEventBridgeClient _eventBridgeClient;

    public Function()
    {
        _eventBridgeClient = new AmazonEventBridgeClient();
    }

    public record Payload(string Key);

    public async Task<APIGatewayProxyResponse> Produce(APIGatewayProxyRequest input, ILambdaContext context)
    {
        var putEventsRequest = new PutEventsRequest
        {
            Entries = new List<PutEventsRequestEntry> 
            {
                new PutEventsRequestEntry
                {
                    Source = "myapplication",
                    DetailType = "mycustomevent",
                    Detail = JsonSerializer.Serialize(new Payload(Guid.NewGuid().ToString())),
                    EventBusName = "MyEventBus"
                }
            }
        };

        var response = await _eventBridgeClient.PutEventsAsync(putEventsRequest);
        if (response.FailedEntryCount > 0)
        {
            return new APIGatewayProxyResponse
            {
                StatusCode = 500,
                Headers = new Dictionary<string, string> { { "Content-Type", "application/json" } }
            };
        }
        else
        {
            return new APIGatewayProxyResponse
            {
                StatusCode = 200,
                Headers = new Dictionary<string, string> { { "Content-Type", "application/json" } }
            };
        }
    }

    public async Task Consume(CloudWatchEvent<Payload> input, ILambdaContext context)
    {
        context.Logger.LogLine("Event Source: " + input.Source);
        context.Logger.LogLine("Event Detail Type: " + input.DetailType);
        context.Logger.LogLine("Event Detail: " + input.Detail.Key);
        await Task.CompletedTask;
    }
}
```

The `Produce` method uses the `AmazonEventBridgeClient` class from the package `AWSSDK.EventBridge` to publish events. Each [`PutEventsRequestEntry`](https://docs.aws.amazon.com/eventbridge/latest/APIReference/API_PutEventsRequestEntry.html#eventbridge-Type-PutEventsRequestEntry-Time) defines:

* `Source`: An identifier for the event's origin.
    
* `DetailType`: The type of event (typically used for filtering).
    
* `Detail`: JSON-serialized data for the event.
    
* `EventBusName`: The name of the event bus. If left empty, the `default` EventBridge will be used.
    

The `Detail`, `DetailType`, and `Source` are required to successfully publish an event. If we include event entries in a request that does not include each of those properties, EventBridge will fail that entry. If we submit a request in which none of the entries have each of these properties, EventBridge fails the entire request.

The `Consume` method is much simpler. It uses the `CloudWatchEvent<T>` class from the `Amazon.Lambda.CloudWatchEvents` package to receive events from EventBridge.

## AWS SAM template

At the solution level, create a `template.yml` file with the following content:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: >
  SAM

Resources:
  MyEventBus: 
    Type: AWS::Events::EventBus
    Properties: 
      Name: "MyEventBus"    

  ProducerFunction:
    Type: AWS::Serverless::Function
    Properties:
      Timeout: 60
      MemorySize: 512
      Tracing: Active
      Runtime: dotnet8
      Architectures:
        - x86_64    
      Handler: MyLambda::MyLambda.Function::Produce
      CodeUri: ./src/MyLambda/
      Policies:
        - EventBridgePutEventsPolicy:
            EventBusName: !Ref MyEventBus
      Events:
        Post:
          Type: Api
          Properties:
            Path: /events
            Method: post

  ConsumerFunction:
    Type: AWS::Serverless::Function
    Properties:
      Timeout: 60
      MemorySize: 512
      Tracing: Active
      Runtime: dotnet8
      Architectures:
        - x86_64    
      Handler: MyLambda::MyLambda.Function::Consume
      CodeUri: ./src/MyLambda/
      Events: 
        Trigger:
          Type: EventBridgeRule
          Properties:
            EventBusName: !Ref MyEventBus
            Pattern:
              source:
                - "myapplication"
              detail-type:
                - "mycustomevent"

Outputs:
  MyApiEndpoint:
    Description: "API endpoint"
    Value: !Sub "https://${ServerlessRestApi}.execute-api.${AWS::Region}.amazonaws.com/Prod/events"
```

The script above defines a custom event bus using the [`AWS::Events::EventBus`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-events-eventbus.html) resource. Then, we defined two functions with the `AWS::Serverless::Function` resource:

* `ProducerFunction`: The producer function uses the [`EventBridgePutEventsPolicy`](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-policy-template-list.html#eventbridge-put-events-policy) policy template, allowing it to publish events to the custom event bus we created earlier.
    
* `ConsumerFunction`: The consumer function uses the [`EventBridgeRule`](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-property-function-eventbridgerule.html) event source type, which sets the function as the target of an Amazon EventBridge rule. The [`Pattern`](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-event-patterns.html) property is used to match the rule with incoming events.
    

> Instead of the `EventBridgeRule` event source, we can use the `AWS::Events::Rule` and `AWS::Lambda::Permission` resources to achieve the same result.

```yaml
  EventBridgeRule:
    Type: AWS::Events::Rule
    Properties:
      Name: MyRule
      EventPattern:
        source:
          - "myapplication"
        detail-type:
          - "mycustomevent"
      State: "ENABLED"
      Targets:
        - Arn: !GetAtt ConsumerFunction.Arn
          Id: "MyTarget"

  LambdaInvokePermission:
    Type: AWS::Lambda::Permission
    Properties:
      FunctionName: !Ref ConsumerFunction
      Action: lambda:InvokeFunction
      Principal: events.amazonaws.com
      SourceArn: !GetAtt MyEventBus.Arn
```

Run the following commands to deploy the resources to AWS:

```powershell
sam build
sam deploy --guided
```

Once deployed, our producer function is ready to be invoked. You can find all the code [here](https://github.com/raulnq/aws-eventbridge). Thanks, and happy coding.