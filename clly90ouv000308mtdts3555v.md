---
title: "Running AWS Lambda Functions Locally Using LocalStack"
datePublished: Wed Aug 30 2023 21:27:53 GMT+0000 (Coordinated Universal Time)
cuid: clly90ouv000308mtdts3555v
slug: running-aws-lambda-functions-locally-using-localstack
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1693344471753/0af7b0a3-1d50-4c95-8c2d-6879445b4f9f.png
tags: aws, net, aws-lambda, aws-sam, localstack

---

In a previous article, [Testing AWS Lambda Functions Locally Using SAM](https://blog.raulnq.com/testing-aws-lambda-functions-locally-using-sam). We discussed how to test Lambda functions on our local machines, simulating triggers from other services. While the AWS SAM local command effectively achieves this goal, one more feature is necessary to execute tests entirely locally: using AWS services such as S3, DynamoDB, SQS, SNS, etc. in our Lambda functions without creating them on AWS. To address this issue, we can use [LocalStack](https://docs.localstack.cloud/overview/):

> [**LocalStack**](https://localstack.cloud/) is a cloud service emulator that runs in a single container on your laptop or in your CI environment. With LocalStack, you can run your AWS applications or Lambdas entirely on your local machine without connecting to a remote cloud provider!

Thus, LocalStack will help us to develop and test our AWS applications (Lambda functions included) locally without incurring costs and latency associated with connecting to remote AWS services. LocalStack enables us to work more efficiently and ensures our applications work correctly before deploying to the AWS environment.

LocalStack supports a wide range of [**AWS services**](https://docs.localstack.cloud/user-guide/aws/), like AWS Lambda, S3, DynamoDB, Kinesis, SQS, SNS, etc. for free. Additionally, there is a paid version called LocalStack Pro, which provides access to extra APIs and advanced features.

## Pre-requisites

* Install the Amazon Lambda Templates with this command: `dotnet new -i Amazon.Lambda.Templates`
    
* Install the Amazon Lambda Tools with this command: `dotnet tool install -g`[`Amazon.Lambda.Tools`](http://Amazon.Lambda.Tools)
    
* Install [Python](https://www.python.org/downloads/windows/).
    
* Install the [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-creds).
    
* Install the [AWS SAM CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html).
    
* Ensure that [**Docker Desktop**](https://docs.docker.com/desktop/install/windows-install/) is up and running.
    

## Installing LocalStack

LocalStack offers multiple [installation](https://docs.localstack.cloud/getting-started/installation/) methods; in this case, we will use the `docker-compose` option. Create a `docker-compose.yml` file with the following content:

```yaml
version: "3.8"
services:
  localstack:
    container_name: "my-localstack"
    image: localstack/localstack
    ports:
      - "127.0.0.1:4566:4566"            # LocalStack Gateway
      - "127.0.0.1:4510-4559:4510-4559"  # external services port range
    environment:
      - DEBUG=1
      - DOCKER_HOST=unix:///var/run/docker.sock
    volumes:
      - ".volume/tmp/localstack:/tmp/localstack"
      - "/var/run/docker.sock:/var/run/docker.sock"
```

Run `docker-compose up`. We can use the [AWS CLI](https://docs.localstack.cloud/user-guide/integrations/aws-cli/) to check our installation by running the command `aws lambda list-functions --endpoint-url=`[`http://localhost:4566`](http://localhost:4566). The `--endpoint-url` specifies the URL to send the request to. We can verify the availability of all services by entering `http://localhost:4566/health` in our web browser.

## The Lambda Function

We will create a Lambda function to register, retrieve, and list tasks. Behind the scenes, a worker will be assigned to each task. All the information will be stored in [DynamoDB](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Introduction.html), and an event will be published to a [SNS](https://docs.aws.amazon.com/sns/latest/dg/welcome.html) topic every time a task is registered. Additionally, an SQS queue will be subscribed to this topic to trigger another Lambda function. Run the following commands to create the project that will host our Lambda functions:

```powershell
dotnet new lambda.EmptyFunction -n MyApp -o .
dotnet add src/MyApp package Amazon.Lambda.APIGatewayEvents
dotnet add src/MyApp package Amazon.Lambda.SQSEvents
dotnet add src/MyApp package AWSSDK.DynamoDBv2
dotnet add src/MyApp package AWSSDK.SimpleNotificationService
dotnet new sln -n LocalStack
dotnet sln add --in-root src/MyApp
```

Open the solution, navigate to the `MyApp` project, and update the `Function.cs` file as follows:

```csharp
using Amazon.DynamoDBv2;
using Amazon.DynamoDBv2.DataModel;
using Amazon.DynamoDBv2.DocumentModel;
using Amazon.DynamoDBv2.Model;
using Amazon.Lambda.APIGatewayEvents;
using Amazon.Lambda.Core;
using Amazon.Lambda.SQSEvents;
using Amazon.SimpleNotificationService;
using Amazon.SimpleNotificationService.Model;
using System.Text.Json;

[assembly: LambdaSerializer(typeof(Amazon.Lambda.Serialization.SystemTextJson.DefaultLambdaJsonSerializer))]
namespace MyApp;

public class Function
{
    private readonly DynamoDBContext _dbContext;
    private readonly AmazonDynamoDBClient _dynamoClient;
    private readonly AmazonSimpleNotificationServiceClient _snsClient;
    private readonly string _tasksSns;

    public Function()
    {
        _dynamoClient = new AmazonDynamoDBClient();
        _dbContext = new DynamoDBContext(_dynamoClient);
        _snsClient = new AmazonSimpleNotificationServiceClient();
        _tasksSns = Environment.GetEnvironmentVariable("TasksSNS")!;
    }

    public async Task<APIGatewayHttpApiV2ProxyResponse> RegisterTask(APIGatewayHttpApiV2ProxyRequest input, ILambdaContext context)
    {
        var req = JsonSerializer.Deserialize<RegisterTaskRequest>(input.Body, new JsonSerializerOptions() { PropertyNameCaseInsensitive = true })!;      
        var task = new TaskModel() { TaskId = Guid.NewGuid(), Name = req.Name };
        await _dbContext.SaveAsync(task);
        var @event = new PublishRequest
        {
            TopicArn = _tasksSns,
            Message = JsonSerializer.Serialize(new TaskRegistered(task.TaskId, task.Name)),
        };
        await _snsClient.PublishAsync(@event);
        return new APIGatewayHttpApiV2ProxyResponse
        {
            Body = JsonSerializer.Serialize(new RegisterTaskResponse(task.TaskId)),
            StatusCode = 200,
            Headers = new Dictionary<string, string> { { "Content-Type", "application/json" } }
        };
    }

    public async Task<APIGatewayHttpApiV2ProxyResponse> GetTask(APIGatewayHttpApiV2ProxyRequest input, ILambdaContext context)
    {
        var taskId = input.PathParameters["taskid"];
        var task = await _dbContext.LoadAsync<TaskModel>(Guid.Parse(taskId));
        if (task == null)
        {
            return new APIGatewayHttpApiV2ProxyResponse
            {
                StatusCode = 404
            };
        }
        var body = JsonSerializer.Serialize(new TaskDTO(task.TaskId, task.Name));
        return new APIGatewayHttpApiV2ProxyResponse
        {
            Body = body,
            StatusCode = 200,
            Headers = new Dictionary<string, string> { { "Content-Type", "application/json" } }
        };
    }

    public async Task<APIGatewayHttpApiV2ProxyResponse> ListTasks(APIGatewayHttpApiV2ProxyRequest input, ILambdaContext context)
    {
        var name = input.QueryStringParameters["name"];
        var request = new ScanRequest()
        {
            TableName = "tasks",
            FilterExpression = "contains(#name, :name)",
            ExpressionAttributeNames = new Dictionary<string, string>()
            {
                { "#name", "name" }
            },
            ExpressionAttributeValues = new Dictionary<string, AttributeValue>()
            {
                {":name", new AttributeValue(name)}
            },
        };

        var response = await _dynamoClient.ScanAsync(request);
        var tasks = response.Items
            .Select(item => Document.FromAttributeMap(item))
            .Select(item => _dbContext.FromDocument<TaskModel>(item));

        var body = JsonSerializer.Serialize(tasks.Select(task=> new TaskDTO(task.TaskId, task.Name)));
        return new APIGatewayHttpApiV2ProxyResponse
        {
            Body = body,
            StatusCode = 200,
            Headers = new Dictionary<string, string> { { "Content-Type", "application/json" } }
        };
    }

    public async Task<SQSBatchResponse> RegisterAssignments(SQSEvent evnt, ILambdaContext context)
    {
        var response = new SQSBatchResponse()
        {
            BatchItemFailures = new List<SQSBatchResponse.BatchItemFailure>()
        };
        foreach (var record in evnt.Records)
        {
            var notification = JsonSerializer.Deserialize<SNSWrapper>(record.Body)!;
            var task = JsonSerializer.Deserialize<TaskRegistered>(notification.Message)!;
            var assignment = new AssingmentModel() { TaskId = task.TaskId, AssignmentId = Guid.NewGuid(), Worker = Guid.NewGuid().ToString() };
            await _dbContext.SaveAsync(assignment);
        }
        return response;
    }

    public async Task<APIGatewayHttpApiV2ProxyResponse> ListAssignments(APIGatewayHttpApiV2ProxyRequest input, ILambdaContext context)
    {
        var taskId = input.PathParameters["taskid"];
        var request = new QueryRequest()
        {
            TableName = "assignments",
            KeyConditionExpression = "taskid = :taskid",
            ExpressionAttributeValues = new Dictionary<string, AttributeValue>()
            {
                {":taskid", new AttributeValue(taskId)}
            },
        };
        var response = await _dynamoClient.QueryAsync(request);
        var assignments = response.Items
            .Select(item => Document.FromAttributeMap(item))
            .Select(item => _dbContext.FromDocument<AssingmentModel>(item));
        var body = JsonSerializer.Serialize(assignments.Select(assignment => new AssingmentDTO(assignment.AssignmentId, assignment.TaskId, assignment.Worker)));
        return new APIGatewayHttpApiV2ProxyResponse
        {
            Body = body,
            StatusCode = 200,
            Headers = new Dictionary<string, string> { { "Content-Type", "application/json" } }
        };
    }

    [DynamoDBTable("tasks")]
    public class TaskModel
    {
        [DynamoDBHashKey("taskid")]
        public Guid TaskId { get; set; }
        [DynamoDBProperty("name")]
        public string? Name { get; set; }
    }

    [DynamoDBTable("assignments")]
    public class AssingmentModel
    {
        [DynamoDBProperty("assignmentid")]
        public Guid AssignmentId { get; set; }
        [DynamoDBHashKey("taskid")]
        public Guid TaskId { get; set; }
        [DynamoDBProperty("worker")]
        public string? Worker { get; set; }
    }

    public class SNSWrapper
    {
        public string Message { get; set; } = null!;
    }

    public record AssingmentDTO(Guid AssignmentId, Guid TaskId, string? Worker);

    public record TaskDTO(Guid TaskId, string? Name);

    public record TaskRegistered(Guid TaskId, string? Name);

    public class RegisterTaskRequest
    {
        public string? Name { get; set; }
    };

    public record RegisterTaskResponse(Guid TaskId);
}
```

Let's explain each function individually:

* **RegisterTask:** The function deserializes the `RegisterTaskRequest` from the request, stores the `TaskModel` into a DynamoDB table, publishes a `TaskRegistered` event to an SNS topic, and returns a `RegisterTaskResponse`.
    
* **GetTask:** The function extracts the `taskid` value from the URL path, loads the `TaskModel` from the DynamoDB table, and returns a `TaskDTO` object.
    
* **ListTasks:** The function extracts the `name` value from the query parameters, scans the DynamoDB table, and returns an array of `TaskDTO` objects.
    
* **RegisterAssignments:** The function deserializes a `TaskRegistered` event, builds an `AssingmentModel`, and stores it in a DynamoDB table.
    
* **ListAssignments:** The function extracts the `taskid` value from the URL path, queries the DynamoDB table, and returns an array of `AssingmentDTO` objects.
    

The Lambda function will be deployed using AWS SAM, so we need to create a `template.yml` file with the following content:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: >
  Local Stack

Globals:
  Function:
    Runtime: dotnet6
    Timeout: 60
    MemorySize: 512
    Architectures:
      - x86_64  

Resources:
  RegisterTaskFunction:
    Type: AWS::Serverless::Function
    Properties:   
      Handler: MyApp::MyApp.Function::RegisterTask
      CodeUri: ./src/MyApp/
      Environment:
        Variables:
          TasksSNS: !Ref TasksSNS
      Policies:
        - DynamoDBCrudPolicy:
            TableName: !Ref TasksTable
        - SNSPublishMessagePolicy:
            TopicName: !Ref TasksSNS
      Events:
        RegisterTask:
          Type: Api
          Properties:
            Path: /api/tasks
            Method: post

  GetTaskFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: MyApp::MyApp.Function::GetTask
      CodeUri: ./src/MyApp/
      Policies:
        - DynamoDBCrudPolicy:
            TableName: !Ref TasksTable
      Events:
        ListTask:
          Type: Api
          Properties:
            Path: /api/tasks/{taskid}
            Method: get

  ListTaskFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: MyApp::MyApp.Function::ListTasks
      CodeUri: ./src/MyApp/
      Policies:
        - DynamoDBCrudPolicy:
            TableName: !Ref TasksTable
      Events:
        ListTask:
          Type: Api
          Properties:
            Path: /api/tasks
            Method: get

  RegisterAssignmentsFunction:
    Type: AWS::Serverless::Function
    Properties:   
      Handler: MyApp::MyApp.Function::RegisterAssignments
      CodeUri: ./src/MyApp/
      Policies:  
        - SQSPollerPolicy:
            QueueName: !GetAtt AssignmentsSQS.QueueName
        - DynamoDBCrudPolicy:
            TableName: !Ref AssignmentsTable
      Events:
        SNSEvent:
          Type: SNS
          Properties:
            Topic: !Ref TasksSNS
            SqsSubscription:
              BatchSize: 10
              QueueArn: !GetAtt AssignmentsSQS.Arn
              QueueUrl: !Ref AssignmentsSQS

  ListAssignmentsFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: MyApp::MyApp.Function::ListAssignments
      CodeUri: ./src/MyApp/
      Policies:
        - DynamoDBCrudPolicy:
            TableName: !Ref AssignmentsTable
      Events:
        ListTask:
          Type: Api
          Properties:
            Path: /api/tasks/{taskid}/assignments
            Method: get

  AssignmentsSQS:
    Type: AWS::SQS::Queue
    Properties:
      QueueName: "assignmentsqueue"

  TasksSNS:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: "taskstopic"

  TasksTable:
    Type: AWS::Serverless::SimpleTable
    Properties:
      PrimaryKey: 
        Name: taskid
        Type: String
      TableName: tasks

  AssignmentsTable:
    Type: AWS::Serverless::SimpleTable
    Properties:
      PrimaryKey: 
        Name: taskid
        Type: String
      TableName: assignments

Outputs:
  ApiId:
    Description: "API Id"
    Value: !Ref ServerlessRestApi
```

The `RegisterTaskFunction`, `GetTaskFunction`, `ListTaskFunction`, and `ListAssignmentsFunction` will be triggered by [API Gateway](https://docs.localstack.cloud/user-guide/aws/apigateway/) endpoints while `RegisterAssignmentsFunction` will be triggered by an [SNS](https://docs.localstack.cloud/user-guide/aws/sns/) event, using an [SQS](https://docs.localstack.cloud/user-guide/aws/sqs/) as a subscription. The TasksTable and AssignmentsTable are our [DynamoDB](https://docs.localstack.cloud/user-guide/aws/dynamodb/) tables.

## Deploying the Lambda Function

LocalStack provides various [integrations](https://docs.localstack.cloud/user-guide/integrations/), and one of them is with [AWS SAM](https://docs.localstack.cloud/user-guide/integrations/aws-sam/). Run the following command:

```powershell
pip install aws-sam-cli-local
```

> The `samlocal` command has the exact same usage as the underlying `sam` command. The main difference is that for commands like `samlocal deploy` the operations will be executed against the LocalStack endpoints ([`http://localhost:4566`](http://localhost:4566) by default) instead of real AWS endpoints.

So, let's run the command `samlocal build`, followed by `samlocal deploy --guided` to deploy our Lambda functions to LocalStack.

## Testing the Lambda Functions

At this point, we have everything in place to begin testing our Lambda function. Let's review all the resources created by the deployment by executing the following commands:

```powershell
aws lambda list-functions --endpoint-url=http://localhost:4566
aws dynamodb list-tables --endpoint-url=http://localhost:4566
aws sqs list-queues --endpoint-url=http://localhost:4566
aws sns list-topics --endpoint-url=http://localhost:4566
aws apigateway get-rest-apis --endpoint-url=http://localhost:4566
```

LocalStack provides a local domain name for our API Gateway endpoints as follows:

```powershell
http://<apiId>.execute-api.localhost.localstack.cloud:4566/<stageId>/<path>
```

We can obtain the `<apiId>` by running the command `aws apigateway get-rest-apis --endpoint-url=http://localhost:4566 --query "items[0].id"`. The `<stageId>` value defaults to `Prod`, and the `<path>` depends on our Lambda function. Let's register a task:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1693427849750/1f00a6e6-0385-4fea-8563-f97b1ae3e2a6.png align="center")

To check if everything worked correctly, run the following command `aws dynamodb scan --table-name tasks --endpoint-url=`[`http://localhost:4566`](http://localhost:4566):

```json
{
    "Items": [
        {
            "name": {
                "S": "my task"
            },
            "taskid": {
                "S": "835398bd-6357-4bb5-8ab0-b17d3a7baea7"
            }
        }
    ],
    "Count": 1,
    "ScannedCount": 1,
    "ConsumedCapacity": null
}
```

The task has been successfully registered. Now, execute the following command `aws dynamodb scan --table-name assignments --endpoint-url=`[`http://localhost:4566`](http://localhost:4566):

```json
{
    "Items": [
        {
            "worker": {
                "S": "0910cb6b-8db3-47ba-bc99-1c621ff3af7c"
            },
            "assignmentid": {
                "S": "3ff85518-2389-4d0c-a062-9ec793e5e99b"
            },
            "taskid": {
                "S": "835398bd-6357-4bb5-8ab0-b17d3a7baea7"
            }
        }
    ],
    "Count": 1,
    "ScannedCount": 1,
    "ConsumedCapacity": null
}
```

Great, the task has been assigned as expected. To clean up our environment, just stop the docker-compose execution, then run the command `docker-compose down`.

In conclusion, LocalStack is a powerful tool that allows developers to emulate AWS services locally, enabling efficient development and testing of Lambda functions without incurring costs or latency. All the code is available [here](https://github.com/raulnq/local-stack). Thanks, and happy coding.