# Deploying AWS Lambda Functions with Serverless Framework

In a previous [post](https://blog.raulnq.com/deploying-aws-lambda-functions-with-terraform), we looked at Terraform as an option to deploy AWS Lambda Function, as long as the number of functions is low. But with a higher number of functions, we could end up building huge scripts. And this is where [Severless Framework](https://www.serverless.com/framework/docs) comes in:

> Serverless Framework is an open source tool available for building, packaging and deploying serverless applications across multiple cloud providers and platforms like AWS, GCP, Azure, Kubernetes, etc

> The Serverless Framework helps you develop and deploy AWS Lambda functions, along with the AWS infrastructure resources they require. It's a CLI that offers structure, automation and best practices out-of-the-box, allowing you to focus on building sophisticated, event-driven, serverless architectures, comprised of Functions and Events.

Before starting, we need to fulfill a few pre-requisites:

- Have a [IAM User](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html#id_users_create_console) with programmatic access.
- [Node](https://nodejs.org/en/) installed on your machine.
- Install the Amazon Lambda Templates (`dotnet new -i Amazon.Lambda.Templates`)
- Install the Amazon Lambda Tools (`dotnet tool install -g Amazon.Lambda.Tools`)

## Installation

```powershell
npm install -g serverless
``` 

## Concepts

- **Services**: A service is the Framework's unit of organization. You can think of it as a project file.
- **Functions**: The code of a serverless application is deployed and executed in AWS Lambda functions.
- **Events**: Functions are triggered by events. Events come from other AWS resources.
- **Resources**: Resources are AWS infrastructure components that your functions use such as a DynamoDB table, an S3 bucket, an SNS topic, etc.
- **Plugins**: You can overwrite or extend the functionality using plugins.

## Let's code

We will create three AWS Lambda Functions to create, get and list tasks. The task data will be stored in a [DynamoBD](https://docs.aws.amazon.com/dynamodb/index.html) table. Run the following command to create the .Net projects and solution:

```powershell
dotnet new lambda.EmptyFunction -n PostTaskFunction -o .
dotnet new lambda.EmptyFunction -n ListTaskFunction -o .
dotnet new lambda.EmptyFunction -n GetTaskFunction -o .
dotnet new classlib -n FunctionLibrary -o src/FunctionLibrary
dotnet new sln -n serverless-framework-sandbox
dotnet sln add --in-root src/PostTaskFunction
dotnet sln add --in-root src/ListTaskFunction
dotnet sln add --in-root src/GetTaskFunction
dotnet sln add --in-root src/FunctionLibrary 
``` 

Add the following NuGet packages to each project:

```powershell
dotnet add src/FunctionLibrary package AWSSDK.DynamoDBv2
dotnet add src/PostTaskFunction package Amazon.Lambda.APIGatewayEvents
dotnet add src/ListTaskFunction package Amazon.Lambda.APIGatewayEvents
dotnet add src/GetTaskFunction package Amazon.Lambda.APIGatewayEvents
``` 

And the references between projects:

```powershell
dotnet add src/PostTaskFunction/PostTaskFunction.csproj reference src/FunctionLibrary/FunctionLibrary.csproj
dotnet add src/ListTaskFunction/ListTaskFunction.csproj reference src/FunctionLibrary/FunctionLibrary.csproj
dotnet add src/GetTaskFunction/GetTaskFunction.csproj reference src/FunctionLibrary/FunctionLibrary.csproj
``` 

Open the solution and under the `FunctionLibrary` project, add the `Task.cs` file with the following content:

```csharp
using Amazon.DynamoDBv2.DataModel;

namespace FunctionLibrary;

[DynamoDBTable("taskstable")]
public class Task
{
    [DynamoDBHashKey("id")]
    public Guid Id { get; set; }
    [DynamoDBProperty("description")]
    public string? Description { get; set; }
    [DynamoDBProperty("title")]
    public string? Title { get; set; }
}
``` 

Under the `PostTaskFunction` project,  modify the `Function.cs` file as follow:

```
using Amazon.DynamoDBv2.DataModel;
using Amazon.DynamoDBv2;
using Amazon.Lambda.APIGatewayEvents;
using Amazon.Lambda.Core;
using System.Text.Json;

[assembly: LambdaSerializer(typeof(Amazon.Lambda.Serialization.SystemTextJson.DefaultLambdaJsonSerializer))]

namespace PostTaskFunction;

public class Function
{
    public record RegisterTaskRequest(string Description, string Title);
    public record RegisterTaskResponse(Guid Id);
    private readonly AmazonDynamoDBClient client;

    private readonly DynamoDBContext dbContext;
    public Function()
    {
        client = new AmazonDynamoDBClient();
        dbContext = new DynamoDBContext(client);
    }

    public async Task<APIGatewayProxyResponse> FunctionHandler(APIGatewayProxyRequest input, ILambdaContext context)
    {
        var req = JsonSerializer.Deserialize<RegisterTaskRequest>(input.Body, new JsonSerializerOptions() { PropertyNameCaseInsensitive = true })!;
        var task = new FunctionLibrary.Task() { Id = Guid.NewGuid(), Description = req.Description, Title = req.Title };
        await dbContext.SaveAsync(task);
        var resp = JsonSerializer.Serialize(new RegisterTaskResponse(task.Id));
        return new APIGatewayProxyResponse
        {
            Body = resp,
            StatusCode = 200,
            Headers = new Dictionary<string, string> { { "Content-Type", "application/json" } }
        };
    }
}
``` 

Under the `ListTaskFunction` project , modify the `Function.cs` file as follow:

```csharp
using Amazon.DynamoDBv2.DataModel;
using Amazon.DynamoDBv2;
using Amazon.Lambda.APIGatewayEvents;
using Amazon.Lambda.Core;
using System.Text.Json;

// Assembly attribute to enable the Lambda function's JSON input to be converted into a .NET class.
[assembly: LambdaSerializer(typeof(Amazon.Lambda.Serialization.SystemTextJson.DefaultLambdaJsonSerializer))]

namespace ListTaskFunction;

public class Function
{
    private readonly AmazonDynamoDBClient client;
    private readonly DynamoDBContext dbContext;
    public Function()
    {
        client = new AmazonDynamoDBClient();
        dbContext = new DynamoDBContext(client);
    }

    public async Task<APIGatewayProxyResponse> FunctionHandler(APIGatewayProxyRequest input, ILambdaContext context)
    {
        var tasks = await dbContext.ScanAsync<FunctionLibrary.Task>(default).GetRemainingAsync();
        var resp = JsonSerializer.Serialize(tasks);
        return new APIGatewayProxyResponse
        {
            Body = resp,
            StatusCode = 200,
            Headers = new Dictionary<string, string> { { "Content-Type", "application/json" } }
        };
    }
}
``` 

Under the `GetTaskFunction` project,  modify the `Function.cs` file as follow:

```csharp
using Amazon.DynamoDBv2.DataModel;
using Amazon.DynamoDBv2;
using Amazon.Lambda.APIGatewayEvents;
using Amazon.Lambda.Core;
using System.Text.Json;

// Assembly attribute to enable the Lambda function's JSON input to be converted into a .NET class.
[assembly: LambdaSerializer(typeof(Amazon.Lambda.Serialization.SystemTextJson.DefaultLambdaJsonSerializer))]

namespace GetTaskFunction;

public class Function
{
    private readonly AmazonDynamoDBClient client;
    private readonly DynamoDBContext dbContext;
    public Function()
    {
        client = new AmazonDynamoDBClient();
        dbContext = new DynamoDBContext(client);
    }

    public async Task<APIGatewayProxyResponse> FunctionHandler(APIGatewayProxyRequest input, ILambdaContext context)
    {
        var id = input.PathParameters["id"];
        var task = await dbContext.LoadAsync<FunctionLibrary.Task>(new Guid(id));
        if (task == null)
        {
            return new APIGatewayProxyResponse
            {
                StatusCode = 404
            };
        }

        var resp = JsonSerializer.Serialize(task);
        return new APIGatewayProxyResponse
        {
            Body = resp,
            StatusCode = 200,
            Headers = new Dictionary<string, string> { { "Content-Type", "application/json" } }
        };
    }
}
``` 

Create the packages to be deployed to AWS with the following commands:

```powershell
dotnet restore src/PostTaskFunction
dotnet lambda package -pl src/PostTaskFunction --configuration Release --framework net6.0 --output-package src/PostTaskFunction/bin/Release/net6.0/PostTaskFunction.zip
dotnet restore src/ListTaskFunction
dotnet lambda package -pl src/ListTaskFunction --configuration Release --framework net6.0 --output-package src/ListTaskFunction/bin/Release/net6.0/ListTaskFunction.zip
dotnet restore src/GetTaskFunction
dotnet lambda package -pl src/GetTaskFunction --configuration Release --framework net6.0 --output-package src/GetTaskFunction/bin/Release/net6.0/GetTaskFunction.zip
``` 

And finally, create a `serverless.yml` file:

```yml
service: task-app
frameworkVersion: '3'

provider:
  name: aws
  stage: ${opt:stage, "dev"}
  region: ${opt:region, "us-east-1"}
  profile: ${opt:profile, "default"}
  runtime: dotnet6
  iam:
    role:
      statements: 
        - Effect: Allow
          Action:
            - dynamodb:*
          Resource: 'arn:aws:dynamodb:us-east-2:*:*'
package:
  individually: true

functions:
  post-tasks:
    handler: PostTaskFunction::PostTaskFunction.Function::FunctionHandler
    package:
      artifact: src/PostTaskFunction/bin/Release/net6.0/PostTaskFunction.zip
    events:
      - http:
          path: /tasks
          method: post
  list-tasks:
    handler: ListTaskFunction::ListTaskFunction.Function::FunctionHandler
    package:
      artifact: src/ListTaskFunction/bin/Release/net6.0/ListTaskFunction.zip
    events:
      - http:
          path: /tasks
          method: get
  get-tasks:
    handler: GetTaskFunction::GetTaskFunction.Function::FunctionHandler
    package:
      artifact: src/GetTaskFunction/bin/Release/net6.0/GetTaskFunction.zip
    events:
      - http:
          path: /tasks/{id}
          method: get

resources:
  Resources:
    tasksTable:
      Type: AWS::DynamoDB::Table
      Properties:
        TableName: taskstable
        AttributeDefinitions:
          - AttributeName: id
            AttributeType: S
        KeySchema:
          - AttributeName: id
            KeyType: HASH
        ProvisionedThroughput:
          ReadCapacityUnits: 1
          WriteCapacityUnits: 1
``` 

Time to deploy. Run `serverless deploy`. The command is going to create three Functions, one AWS API Gateway, and one DynamoDB Table:

```
Deploying task-app to stage dev (us-east-1)

✔ Service deployed to stack task-app-dev (124s)

endpoints:
  POST - https://y4l82ho3d5.execute-api.us-east-1.amazonaws.com/dev/tasks
  GET - https://y4l82ho3d5.execute-api.us-east-1.amazonaws.com/dev/tasks
  GET - https://y4l82ho3d5.execute-api.us-east-1.amazonaws.com/dev/tasks/{id}
functions:
  post-tasks: task-app-dev-post-tasks (1.7 MB)
  list-tasks: task-app-dev-list-tasks (1.7 MB)
  get-tasks: task-app-dev-get-tasks (1.7 MB)
``` 

To clean up all the resources run `serverless remove`. For more details and other command execution options, take a look at the Serverless Framework [documentation](https://www.serverless.com/framework/docs/providers/aws/guide/serverless.yml).
All the code is available [here](https://github.com/raulnq/serverless-framework-sandbox). Thanks, and happy coding.
