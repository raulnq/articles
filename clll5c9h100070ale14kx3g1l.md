---
title: "Testing AWS Lambda Functions Locally Using SAM"
datePublished: Mon Aug 21 2023 17:23:54 GMT+0000 (Coordinated Universal Time)
cuid: clll5c9h100070ale14kx3g1l
slug: testing-aws-lambda-functions-locally-using-sam
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1692403793844/1549ef63-7c06-4f91-a0ce-70daaf7f3154.png
tags: aws, net, aws-sam

---

Testing AWS Lambda functions locally can be a crucial step in the development process, as it allows developers to identify and resolve issues before deploying their code to the cloud. In this regard, AWS SAM provides a method for testing our API called `sam local`, which enables us to perform tasks such as:

* Run AWS Lambda functions locally.
    
* Test API Gateway endpoints.
    
* Simulate event triggers from services like S3, SNS, and SQS.
    
* And even debug AWS Lambda functions.
    

All this works using Docker containers to replicate the AWS runtime environment on your local machine. When we issue a `sam local` command, SAM reads your template file, pulls the appropriate Docker image for the specified Lambda runtime, then packages our code, dependencies, and environment variables into a Docker container, simulating the actual AWS Lambda environment as closely as possible.

## Pre-requisites

* Ensure you have an [**IAM User**](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html#id_users_create_console) with programmatic access.
    
* Install the Amazon Lambda Templates with this command: `dotnet new -i Amazon.Lambda.Templates`
    
* Install the Amazon Lambda Tools with this command: `dotnet tool install -g` [`Amazon.Lambda.Tools`](http://Amazon.Lambda.Tools)
    
* Install the [**AWS SAM CLI**](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html).
    
* Ensure that [**Docker Desktop**](https://docs.docker.com/desktop/install/windows-install/) is up and running.
    

## The Lambda Function

Run the following commands to create the project that will host our Lambda functions:

```powershell
dotnet new lambda.EmptyFunction -n MyLambdaFunctions -o .
dotnet add src/MyLambdaFunctions package Amazon.Lambda.APIGatewayEvents
dotnet add src/MyLambdaFunctions package Amazon.Lambda.SQSEvents
dotnet new sln -n SamLocal
dotnet sln add --in-root src/MyLambdaFunctions 
```

Open the solution, navigate to the `MyLambdaFunctions` project, and update the `Function.cs` file as follows:

```csharp
using Amazon.Lambda.APIGatewayEvents;
using Amazon.Lambda.Core;
using Amazon.Lambda.SQSEvents;

[assembly: LambdaSerializer(typeof(Amazon.Lambda.Serialization.SystemTextJson.DefaultLambdaJsonSerializer))]
namespace MyLambdaFunctions;

public class Function
{
    public string FunctionHandler(string input, ILambdaContext context)
    {
        context.Logger.LogInformation($"new message: {input}");

        return "Hello world";
    }

    public APIGatewayHttpApiV2ProxyResponse ApiFunctionHandler(APIGatewayHttpApiV2ProxyRequest input, ILambdaContext context)
    {
        context.Logger.LogInformation($"new message: {input.Body}");

        return new APIGatewayHttpApiV2ProxyResponse
        {
            Body = @"{""Message"":""Hello World""}",
            StatusCode = 200,
            Headers = new Dictionary<string, string> { { "Content-Type", "application/json" } }
        };
    }

    public void MessageFunctionHandler(SQSEvent evnt, ILambdaContext context)
    {
        foreach (var record in evnt.Records)
        {
            context.Logger.LogInformation($"new message: {record.Body}");
        }
    }
}
```

We are defining three Lambda functions, two of which have a request and a message as triggers. Create a `template.yml` file with the following content:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: >
  SAM Local

Resources:
  MyFunction:
    Type: AWS::Serverless::Function
    Properties:
      Timeout: 60
      MemorySize: 512
      Tracing: Active
      Runtime: dotnet6
      Architectures:
        - x86_64    
      Handler: MyLambdaFunctions::MyLambdaFunctions.Function::FunctionHandler
      CodeUri: ./src/MyLambdaFunctions/

  MyApiFunction:
    Type: AWS::Serverless::Function
    Properties:
      Timeout: 60
      MemorySize: 512
      Tracing: Active
      Runtime: dotnet6
      Architectures:
        - x86_64    
      Handler: MyLambdaFunctions::MyLambdaFunctions.Function::ApiFunctionHandler
      CodeUri: ./src/MyLambdaFunctions/
      Events:
        ListPosts:
          Type: Api
          Properties:
            Path: /api
            Method: get

  MyMessageFunction:
    Type: AWS::Serverless::Function
    Properties:
      Runtime: dotnet6
      Handler: MyLambdaFunctions::MyLambdaFunctions.Function::MessageFunctionHandler
      CodeUri: ./src/MyLambdaFunctions/
      Policies:  
        - SQSPollerPolicy:
            QueueName: !GetAtt SQSQueue.QueueName
      Events:
        SQSEvent:
          Type: SQS
          Properties:
            Queue: !GetAtt SQSQueue.Arn
            BatchSize: 10

  SQSQueue:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: "mySQSQueue"
```

Run `sam build` to build the resources declared in the template. This must be done every time changes are made.

## Invoke a Lambda Function Locally

The standard [syntax](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/using-sam-cli-local-invoke.html) of the command is:

```powershell
sam local invoke <functionlogicalid> --event <file>
```

Create a folder named `events` and create a file called `myfunction.txt` with the following content:

```yaml
"My lambda function input"
```

Run `sam local invoke MyFunction --event .\events\myfunction.txt` to see the following output:

```powershell
START RequestId: a996bd93-68f1-4011-867d-9646ec188542 Version: $LATEST
2023-08-21T00:52:01.317Z        a996bd93-68f1-4011-867d-9646ec188542    info    new message: My lambda function input
END RequestId: a996bd93-68f1-4011-867d-9646ec188542
REPORT RequestId: a996bd93-68f1-4011-867d-9646ec188542  Init Duration: 0.09 ms  Duration: 195.77 ms     Billed Duration: 196 ms Memory Size: 512 MB   Max Memory Used: 512 MB
"Hello world"
```

## Generate an Event to Invoke a Lambda Function

We can use `sam local generate-event` command to generate events for supported AWS services. We can run `sam local generate-event` to display the list of supported AWS services. The standard [syntax](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/using-sam-cli-local-generate-event.html) of the command is:

```powershell
sam local generate-event <service> <event>
```

For example, by running the command `sam local generate-event apigateway -h`, we can see all the events available for the API Gateway service:

```powershell
Options:
  -h, --help  Show this message and exit.

Commands:
  authorizer          Generates an Amazon API Gateway Authorizer Event
  aws-proxy           Generates an Amazon API Gateway AWS Proxy Event
  http-api-proxy      Generates an Amazon API Gateway Http API Event
  request-authorizer  Generates an Amazon API Gateway Request Authorizer Event
```

And, by running `sam local generate-event apigateway http-api-proxy -h`, we can see different sections of the event that can be modified:

```powershell
Options:
  --body TEXT         Specify the body name you'd like, otherwise the default
                      = {"test":"body"}
  --stage TEXT        Specify the stage name you'd like, otherwise the default
                      = $default
...
```

Run `sam local generate-event apigateway http-api-proxy > .\events\myapifunction.json` and **ensure the generated file is in** `UTF-8` **encoding**. Now, run `sam local invoke MyApiFunction --event .\events\myapifunction.json` to see the following output:

```powershell
START RequestId: 40b6d72f-6c3d-4e05-ac1c-9f8d94a5662c Version: $LATEST
2023-08-21T01:19:15.296Z        40b6d72f-6c3d-4e05-ac1c-9f8d94a5662c    info    new message: eyJ0ZXN0IjoiYm9keSJ9
END RequestId: 40b6d72f-6c3d-4e05-ac1c-9f8d94a5662c
REPORT RequestId: 40b6d72f-6c3d-4e05-ac1c-9f8d94a5662c  Init Duration: 0.19 ms  Duration: 248.69 ms     Billed Duration: 249 ms Memory Size: 512 MB   Max Memory Used: 512 MB
{"statusCode":200,"headers":{"Content-Type":"application/json"},"body":"{{\"Message\":\"Hello World\"}}","isBase64Encoded":false}
```

The same exercise can be performed for our third Lambda function, run `sam local generate-event sqs receive-message --body "Hi" > .\events\mymessagefunction.json` and then `sam local invoke MyMessageFunction --event .\events\mymessagefunction.json`:

```powershell
Mounting C:\Source\sam-local\.aws-sam\build\MyMessageFunction as /var/task:ro,delegated inside runtime container
START RequestId: ab673232-20c0-407d-8412-c49213b48566 Version: $LATEST
2023-08-21T01:26:22.898Z        ab673232-20c0-407d-8412-c49213b48566    info    new message: Hi
END RequestId: ab673232-20c0-407d-8412-c49213b48566
REPORT RequestId: ab673232-20c0-407d-8412-c49213b48566  Init Duration: 0.26 ms  Duration: 187.51 ms     Billed Duration: 188 ms Memory Size: 128 MB   Max Memory Used: 128 MB
```

## Starting a Local API Gateway

The standard [syntax](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/using-sam-cli-local-start-api.html) of the command is:

```powershell
sam local start-api <options>
```

One of the most useful options is `--warm-containers`:

* **eager:** Containers for all functions are loaded at startup and persist between invocations.
    
* **lazy**: Containers are only loaded when each function is first invoked and persisted for additional invocations.
    

Without this option, the command will create a new container each time our function is invoked. Run `sam local start-api --warm-containers lazy` and invoke the Lambda function through the browser:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1692582069146/b9828273-a763-4a86-8c50-d77a81ad5c5b.png align="center")

All the code is available [here](https://github.com/raulnq/sam-local). Thanks, and happy coding.