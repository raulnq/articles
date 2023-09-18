---
title: "Send Amazon CloudWatch Alarms to Slack"
datePublished: Mon Sep 18 2023 14:34:46 GMT+0000 (Coordinated Universal Time)
cuid: clmozmlsi000109mn1bijf0wf
slug: send-amazon-cloudwatch-alarms-to-slack
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1694819220510/becc5241-be9c-4c8d-876c-05816d8e80c5.png
tags: slack, net, aws-lambda, localstack, nuke

---

The following article provides a practical example of [Running AWS Lambda Functions Locally Using LocalStack](https://blog.raulnq.com/running-aws-lambda-functions-locally-using-localstack). The purpose of this AWS Lambda function is to forward AWS CloudWatch alarms to Slack by utilizing an SNS topic as an intermediary. Please **ensure you have completed all the prerequisites mentioned in the referenced article** before continuing. In addition, we will use [Nuke](https://nuke.build/) to automate various helpful tasks. You can learn more about Nuke [here](https://blog.raulnq.com/series/nuke). So, let's begin.

## Lambda Function

Run the following commands to set up our project:

```powershell
dotnet new lambda.EmptyFunction -n SNS2Slack -o .
dotnet add src/SNS2Slack package Slack.Webhooks
dotnet add src/SNS2Slack package Amazon.Lambda.SNSEvents
dotnet new sln -n SNS2Slack
dotnet sln add --in-root src/SNS2Slack
```

Open the solution, navigate to the `SNS2Slack` project, and create an `Alarm.cs` file containing the following content:

```csharp
namespace SNS2Slack;

public class Alarm
{
    public string? AlarmName { get; set; }
    public string? AlarmDescription { get; set; }
    public string? AWSAccountId { get; set; }
    public string? NewStateValue { get; set; }
    public string? NewStateReason { get; set; }
    public string? OldStateValue { get; set; }
    public Trigger? Trigger { get; set; }
}

public class Trigger
{
    public string? MetricName { get; set; }
    public string? Namespace { get; set; }
    public string? Statistic { get; set; }
    public string? GreaterThanOrEqualToThreshold { get; set; }
    public decimal Period { get; set; }
    public decimal Threshold { get; set; }
    public decimal EvaluationPeriods { get; set; }
}
```

Open the `Function.cs` file and update the content as follows:

```csharp
using Amazon.Lambda.Core;
using Amazon.Lambda.SNSEvents;
using Slack.Webhooks.Elements;
using Slack.Webhooks;
using System.Text.Json;
using Slack.Webhooks.Blocks;
[assembly: LambdaSerializer(typeof(Amazon.Lambda.Serialization.SystemTextJson.DefaultLambdaJsonSerializer))]
namespace SNS2Slack;
public class Function
{
    private readonly SlackClient _slackClient;
    public Function()
    {
        var webHook = Environment.GetEnvironmentVariable("SlackWebHook");
        _slackClient = new SlackClient(webHook);
    }

    public async Task FunctionHandler(SNSEvent evnt, ILambdaContext context)
    {
        foreach (var record in evnt.Records)
        {
            await ProcessRecord(record, context);
        }
    }

    private async Task ProcessRecord(SNSEvent.SNSRecord record, ILambdaContext context)
    {
        context.Logger.LogInformation($"Processed record {record.Sns.Message}");
        if (record.Sns.Message == null)
        {
            return;
        }
        var alarm = JsonSerializer.Deserialize<Alarm>(record.Sns.Message);
        if (string.IsNullOrEmpty(alarm?.AlarmName))
        {
            return;
        }
        var slackMessage = new SlackMessage
        {
            Markdown = true
        };
        slackMessage.Blocks = new List<Block>
        {
            new Section
            {
                Text = new TextObject($"{alarm.AlarmDescription}\n")
                {
                    Type = TextObject.TextType.Markdown
                }
            }
        };
        await _slackClient.PostAsync(slackMessage);
    }
}
```

Essentially, we receive the alarm within an SNS record and subsequently send its description to Slack. Create a `template.yml` file as follows:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: >
  SNS

Globals:
  Function:
    Timeout: 15
    MemorySize: 512
    Runtime: dotnet6
    Architectures:
      - x86_64

Parameters:
  SlackWebHook:
    Type: String

Resources:
  SNSTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: "SlackTopic"

  SlackForwarderFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: SNS2Slack::SNS2Slack.Function::FunctionHandler
      CodeUri: ./src/SNS2Slack/
      Environment:
        Variables:
          SlackWebHook: !Ref SlackWebHook
      Events:
        SNSEvent:
          Type: SNS
          Properties:
            Topic: !Ref SNSTopic

Outputs:
  SNSArn:
    Description: SNS ARN
    Value: !Ref SNSTopic
```

The SAM script creates an SNS topic to which CloudWatch Alarms will be sent and a Lambda function triggered by the mentioned SNS topic.

## Local Stack

Add a `docker-compose.yml` file at the solution level with the following content:

```yaml
version: "3.8"

services:
  localstack:
    container_name: "my-localstack"
    image: localstack/localstack
    ports:
      - "127.0.0.1:4566:4566"
      - "127.0.0.1:4510-4559:4510-4559"
    environment:
      - DEBUG=1
      - DOCKER_HOST=unix:///var/run/docker.sock
    volumes:
      - ".volume/tmp/localstack:/tmp/localstack"
      - "/var/run/docker.sock:
```

## Slack

Go to [https://api.slack.com/apps](https://api.slack.com/apps) and create a new application (from scratch):

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1695041342912/31b30ec4-8f9a-4902-9932-3f9b2eb37ea0.png align="center")

Activate the `Incoming WebHooks` feature:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1695041412030/530634db-93b1-4731-8671-99cfda764319.png align="center")

Finally, add a new Webhook to Workspace:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1695041519060/b89c7a32-7b3a-488f-9698-895121ce6ac5.png align="center")

Once completed, copy the Webhook URL for future use.

## Nuke

Run the `nuke :setup` command and follow the instructions (we recommend using all the default values) to set up the project. Navigate to the `_build` the project and update the content as follows:

```csharp
using Nuke.Common;
using Nuke.Common.Tooling;

class Build : NukeBuild
{
    [PathExecutable(name: "docker-compose")]
    public readonly Tool DockerCompose;
    [PathExecutable(name: "samlocal")]
    public readonly Tool SamLocal;
    public static int Main () => Execute<Build>(x => x.SamLocalBuild);
    [Parameter()]
    public string SlackWebHook;
    public bool IsRunning()
    {
        var output = DockerCompose("ps --status running --quiet");
        return output.Count > 0;
    }

    Target StartEnv => _ => _
    .OnlyWhenDynamic(() => !IsRunning())
    .Executes(() =>
    {
        DockerCompose("up -d");
    });

    Target StopEnv => _ => _
        .Executes(() =>
        {
            DockerCompose("down");
        });

    Target SamLocalBuild => _ => _
        .Executes(() =>
        {
            SamLocal("build");
        });

    Target SamLocalDeploy => _ => _
        .Requires(() => SlackWebHook)
        .DependsOn(SamLocalBuild)
        .DependsOn(StartEnv)
        .Executes(() =>
        {
            SamLocal($"deploy --no-confirm-changeset --disable-rollback --resolve-s3 --s3-prefix sns2slack --stack-name sns2slack --region us-east-1  --capabilities CAPABILITY_IAM --parameter-overrides SlackWebHook={SlackWebHook}");
        });
}
```

In the class above, we are automating the following tasks:

* `StartEnv`: Start the LocalStack environment by running the docker-compose file.
    
* `SamLocalBuild`: Build the Lambda function using `samlocal` CLI.
    
* `SamLocalDeploy`: Deploy the Lambda function to the LocalStack environment, requesting the Slack Webhook URL.
    
* `StopEnv`: Stop and remove the LocalStack environment.
    

Run the following command to deploy the Lambda function locally:

```powershell
nuke SamLocalDeploy --slack-web-hook <MY_WEBHOOK_URL>
```

We can use `aws lambda list-functions --endpoint-url=`[`http://localhost:4566`](http://localhost:4566) to verify the Lambda function deployment.

## Cloud Watch Alarm

Register a CloudWatch alarm by using the created SNS topic as an action:

```powershell
aws cloudwatch put-metric-alarm --endpoint-url=http://localhost:4566 --alarm-name my-alarm --alarm-description 'This is my slack message' --comparison-operator GreaterThanThreshold --evaluation-periods 1 --alarm-actions arn:aws:sns:us-east-1:000000000000:SlackTopic
```

Next, change the state of the alarm with:

```powershell
aws cloudwatch set-alarm-state --endpoint-url=http://localhost:4566 --alarm-name my-alarm --state-reason "Testing" --state-value ALARM
```

In conclusion, this article demonstrates how to send Amazon CloudWatch alarms to Slack using AWS Lambda functions. By following the provided steps, we'll be able to set up and deploy a Lambda function locally **(**thanks to LocalStack), integrate it with Slack, and automate useful tasks with Nuke. All the code is available [here](https://github.com/raulnq/aws-lambda-slack). Thanks, and happy coding.