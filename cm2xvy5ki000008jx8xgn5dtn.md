---
title: "Getting Started with Amazon EventBridge Scheduler for .NET Developers"
datePublished: Thu Oct 31 2024 22:37:00 GMT+0000 (Coordinated Universal Time)
cuid: cm2xvy5ki000008jx8xgn5dtn
slug: getting-started-with-amazon-eventbridge-scheduler-for-net-developers
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1730243121764/127a7e6d-5e5c-4c2c-a147-02fc39ed40f9.png
tags: aws, dotnet, aws-lambda, scheduler, aws-eventbridge

---

In our previous article, [Getting Started with Amazon EventBridge for .NET Developers](https://blog.raulnq.com/getting-started-with-amazon-eventbridge-for-net-developers), we delved into the capabilities of [Amazon EventBridge](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-what-is.html) for routing events through applications. But what if we need to trigger an event based on time? To address this, Amazon EventBridge provides [schedule rules](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-create-rule-schedule.html). However, the currently recommended approach is to use the [Amazon EventBridge Scheduler](https://docs.aws.amazon.com/scheduler/latest/UserGuide/what-is-scheduler.html).

> Amazon EventBridge Scheduler is a serverless scheduler that allows you to create, run, and manage tasks from one central managed service.

Amazon EventBridge Scheduler supports several types of [scheduling patterns](https://docs.aws.amazon.com/scheduler/latest/UserGuide/schedule-types.html):

* Rate-based: This type allows tasks to run at regular intervals. The rate expression is `rate(value unit)`, where `unit` can be minutes, hours, or days.
    
* Cron-based: This type allows tasks to run based on a cron expression, allowing for more detailed schedules. The cron expression is `cron(minutes hours day-of-month month day-of-week year)`. Rate-based and cron-based schedules can be triggered in a specific timeframe using a start date and an end date.
    
* One-time: A one-time schedule triggers a target only once at the specific date and time we set. The one-time expression is `at(yyyy-mm-ddThh:mm:ss)`.
    

> All schedule types on EventBridge Scheduler invoke their targets with 60 second precision. This means that if you set your schedule to run at `1:00`, it will invoke the target API between `1:00:00` and `1:00:59`, assuming that a flexible time window is not set.

In Amazon EventBridge Scheduler, we can use numerous targets that are divided into two groups:

* [Templated targets](https://docs.aws.amazon.com/scheduler/latest/UserGuide/managing-targets-templated.html): These are a predefined set of API operations against a group of AWS services. To use them, we need to provide the `Arn` of the service, the `RoleArn` that Amazon EventBridge Scheduler will assume to trigger the target, and optionally an `input` (the event payload).
    
* [Universal targets](https://docs.aws.amazon.com/scheduler/latest/UserGuide/managing-targets-universal.html): These types of targets allow you to invoke a wider set of API operations. The parameters are similar, but the content differs slightly:
    
    * `Arn`: The complete service ARN, including the API operation we want to target. `arn:aws:scheduler:::aws-sdk:service:apiAction`
        
    * `Input`: In this case, it is not just the event payload; it is the entire payload for the target API.
        

Other features worth mentioning are:

* [Flexible Time Window](https://docs.aws.amazon.com/scheduler/latest/UserGuide/managing-schedule-flexible-time-windows.html): this setting allows events to be triggered within a specified time range instead of at an exact, fixed time.
    
* **Error-handling**: Amazon EventBridge Scheduler lets us set the number of retries for our schedule. If we reach the maximum number of retries, it supports [Dead-Letter Queues](https://docs.aws.amazon.com/scheduler/latest/UserGuide/configuring-schedule-dlq.html).
    
* **Time Zones**: Amazon EventBridge Scheduler offers timezone support to ensure events are triggered at the correct local times.
    

Let's put everything into practice by developing a one-time schedule and a rate-based schedule, both targeting a Lambda function.

## Pre-requisites

* Have an [**IAM User**](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html#id_users_create_console) with programmatic access.
    
* Install the Amazon Lambda Templates (`dotnet new -i Amazon.Lambda.Templates`)
    
* Install the Amazon Lambda Tools (`dotnet tool install -g Amazon.Lambda.Tools`)
    
* Install [**AWS SAM CLI**](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html).
    

## **The Backend Services**

Run the following commands to set up our Lambda functions:

```powershell
dotnet new lambda.EmptyFunction -n MyLambda -o .
dotnet add src/MyLambda package AWSSDK.Scheduler
dotnet add src/MyLambda package Amazon.Lambda.APIGatewayEvents
dotnet new sln -n EventBridgeScheduler
dotnet sln add --in-root src/MyLambda
```

Open the `Program.cs` file and update the content as follows:

```csharp
using Amazon.Scheduler;
using Amazon.Scheduler.Model;
using Amazon.Lambda.APIGatewayEvents;
using Amazon.Lambda.Core;
using System.Text.Json;
using System.Globalization;

[assembly: LambdaSerializer(typeof(Amazon.Lambda.Serialization.SystemTextJson.DefaultLambdaJsonSerializer))]
namespace MyLambda;

public class Function
{
    private readonly AmazonSchedulerClient _schedulerClient;
    private readonly string _targetArn;
    private readonly string _roleArn;

    public Function()
    {
        _schedulerClient = new AmazonSchedulerClient();
        _targetArn = Environment.GetEnvironmentVariable("TARGET_ARN")!;
        _roleArn = Environment.GetEnvironmentVariable("ROLE_ARN")!;
    }

    public record Payload(string Key);

    public async Task<APIGatewayProxyResponse> Produce(APIGatewayProxyRequest input, ILambdaContext context)
    {
        var request = new CreateScheduleRequest
        {
            Name = $"myschedule-{input.RequestContext.RequestId}",
            ScheduleExpression = $"at({DateTime.UtcNow.AddMinutes(5).ToString("yyyy-MM-ddTHH:mm:ss", CultureInfo.InvariantCulture)})",
            GroupName = "myapp",
            State = ScheduleState.ENABLED,
            Target = new Target { Arn = _targetArn, RoleArn = _roleArn, Input = JsonSerializer.Serialize(new Payload(Guid.NewGuid().ToString())) },
            ActionAfterCompletion = ActionAfterCompletion.DELETE,
            FlexibleTimeWindow = new FlexibleTimeWindow
            {
                Mode = FlexibleTimeWindowMode.OFF,
            }
        };

        await _schedulerClient.CreateScheduleAsync(request);

        return new APIGatewayProxyResponse
        {
            StatusCode = 200,
            Headers = new Dictionary<string, string> { { "Content-Type", "application/json" } }
        };
    }

    public async Task Consume(Payload input, ILambdaContext context)
    {
        context.Logger.LogLine("Key: " + input.Key);
        await Task.CompletedTask;
    }
}
```

The `Produce` method uses the `AmazonSchedulerClient` class from the package `AWSSDK.Scheduler` to create a one-time schedule. On the other hand, the `Consume` method accepts an object of the `Payload` class as a parameter, which is the same class used to serialize the input in the previous method.

## AWS SAM template

At the solution level, create a `template.yml` file with the following content:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: >
  SAM

Resources:
  MyEventBus: 
    Type: AWS::Scheduler::ScheduleGroup
    Properties: 
      Name: "myapp"    

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

  ConsumerRole:
    Type: AWS::IAM::Role
    Properties:
      RoleName: ConsumerRole
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              Service: "scheduler.amazonaws.com"
            Action: "sts:AssumeRole"
      Policies:
        - PolicyName: AllowInvokeLambda
          PolicyDocument:
            Version: "2012-10-17"
            Statement:
              - Effect: Allow
                Action: 
                  - "lambda:InvokeFunction"
                Resource: 
                  - !GetAtt ConsumerFunction.Arn

  ProducerFunction:
    Type: AWS::Serverless::Function
    Properties:
      Timeout: 60
      Environment:
        Variables:
          TARGET_ARN: !GetAtt ConsumerFunction.Arn
          ROLE_ARN: !GetAtt ConsumerRole.Arn
      MemorySize: 512
      Tracing: Active
      Runtime: dotnet8
      Architectures:
        - x86_64    
      Handler: MyLambda::MyLambda.Function::Produce
      CodeUri: ./src/MyLambda/
      Policies:
        - Version: '2012-10-17' 
          Statement:
          - Effect: Allow
            Action:
              - scheduler:CreateSchedule
            Resource: '*'
          - Effect: Allow
            Action: 
              - iam:PassRole
            Resource: !GetAtt ConsumerRole.Arn
      Events:
        Post:
          Type: Api
          Properties:
            Path: /events
            Method: post

  RateSchedule:
    Type: AWS::Scheduler::Schedule
    Properties:
      GroupName: myapp
      Name: "myrateschedule"
      State: ENABLED
      FlexibleTimeWindow:
        Mode: 'OFF'
      ScheduleExpression: "rate(5 minutes)"
      Target:
        Arn: !GetAtt ConsumerFunction.Arn
        RoleArn: !GetAtt ConsumerRole.Arn
        Input: '{"Key": "123"}'
        RetryPolicy:
          MaximumRetryAttempts: 10

  DailyCronSchedule:
    Type: AWS::Scheduler::Schedule
    Properties:
      GroupName: myapp
      Name: "mydailycronschedule"
      State: ENABLED
      FlexibleTimeWindow:
        Mode: 'FLEXIBLE'
        MaximumWindowInMinutes: 10
      ScheduleExpression: "cron(15 10 * * ? *)"
      Target:
        Arn: !GetAtt ConsumerFunction.Arn
        RoleArn: !GetAtt ConsumerRole.Arn
        Input: '{"Key": "abc"}'

Outputs:
  MyApiEndpoint:
    Description: "API endpoint"
    Value: !Sub "https://${ServerlessRestApi}.execute-api.${AWS::Region}.amazonaws.com/Prod/events"
```

The script above defines a new group with the [`AWS::Scheduler::ScheduleGroup`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-scheduler-schedulegroup.html) resource. To trigger the `ConsumerFunction` from Amazon EventBridge Scheduler, we define a resource of type `AWS::IAM::Role` with the appropriate permissions. The `ProducerFunction` requires two permissions:

* `scheduler:CreateSchedule` to create the schedule.
    
* `iam:PassRole` to pass the role to the service.
    

In addition to the resources needed to create the one-time schedule, we included the creation of the `RateSchedule` and `DailyCronSchedule` using the [`AWS::Scheduler::Schedule`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-scheduler-schedule.html) resource. The most common properties under this resource are:

* `GroupName`: The name of the schedule group associated with this schedule.
    
* `Name`: The name of the schedule.
    
* `State`: Specifies whether the schedule is enabled or disabled.
    
* `ScheduledExpression`: This expression defines when the schedule runs.
    
* [`FlexibleTimeWindow`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-scheduler-schedule-flexibletimewindow.html): This allows you to set up a time window during which Amazon EventBridge Scheduler will trigger the schedule.
    
* [`Target`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-scheduler-schedule-target.html): The details of the schedule's target.
    

Run the following commands to deploy the resources to AWS:

```powershell
sam build
sam deploy --guided
```

Optionally, we can define rate-based and cron-based schedules using the [`ScheduleV2`](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-property-function-schedulev2.html) event source within the `AWS::Serverless::Function` resource itself:

```yaml
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
        ScheduleEvent:
          Type: ScheduleV2
          Properties:
            ScheduleExpression: "rate(10 minute)"
            Name: "myrateschedulev2"
            GroupName: myapp
            State: ENABLED
            Input: '{"Key": "xyz"}'
            RetryPolicy:
              MaximumRetryAttempts: 5
            DeadLetterConfig:
              Type: SQS
```

You can find all the code [here](https://github.com/raulnq/aws-eventbridge-scheduler). Thanks, and happy coding.