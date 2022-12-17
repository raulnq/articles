# Deploying AWS Lambda Functions with AWS SAM

Working with serverless technologies like Lambda Functions is great, but it is easy to lose control of what to deploy (due to the large number of resources we usually have to deploy). In a previous post, we explore a solution for that, [Serverless Framework](https://blog.raulnq.com/deploying-aws-lambda-functions-with-serverless-framework), and today we will try [AWS Serverless Application Model (SAM)](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/what-is-sam.html).

> The AWS Serverless Application Model (AWS SAM) is an open-source framework for building serverless applications. It provides shorthand syntax to express functions, APIs, databases, and event source mappings. With just a few lines per resource, you can define the application you want and model it using YAML. During deployment, AWS SAM transforms and expands the SAM syntax into AWS CloudFormation syntax, enabling you to save time and build, test, and deploy serverless applications faster.

## Pre-requisites

*   Have an [IAM User](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html#id_users_create_console) with programmatic access. 
*   Install the Amazon Lambda Templates (`dotnet new -i Amazon.Lambda.Templates`)
*   Install the Amazon Lambda Tools (`dotnet tool install -g Amazon.Lambda.Tools`)
*   Install [AWS SAM CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html).

## Let's see the code

We developed three AWS Lambda Functions to create, get and list tasks (the task data will be stored in a [DynamoBD](https://docs.aws.amazon.com/dynamodb/index.html) table). One of the reasons to try AWS SAM is the support to deploy .NET 7 applications, which allows us to use Native AOT compilation (to reduce the cold start times). Download the solution [here](https://github.com/raulnq/aws-sam) and open it. Go to the `FunctionLibrary` project and open the `TaskRepository.cs` file:

```csharp
public class TasksRepository
{
    private AmazonDynamoDBClient _amazonDynamoDB;
    private string _tableName;

    public TasksRepository(AmazonDynamoDBClient amazonDynamoDB, string tableName)
    {
        _amazonDynamoDB = amazonDynamoDB;
        _tableName = tableName;
    }

    public System.Threading.Tasks.Task Save(Task task)
    {
        var putItemRequest = new PutItemRequest
        {
            TableName = _tableName,
            Item = new Dictionary<string, AttributeValue> {
                    {
                        "id",
                        new AttributeValue {
                        S = task.Id.ToString(),
                    }
                    },
                    {
                        "description",
                        new AttributeValue {
                        S = task.Description
                        }
                    },
                    {
                        "title",
                        new AttributeValue {
                        S = task.Title
                        }
                    }
                }
        };

        return _amazonDynamoDB.PutItemAsync(putItemRequest);
    }

    public async Task<Task?> Get(Guid id)
    {
        var request = new GetItemRequest
        {
            TableName = _tableName,
            Key = new Dictionary<string, AttributeValue>() { { "id", new AttributeValue { S = id.ToString() } } },
        };

        var response = await _amazonDynamoDB.GetItemAsync(request);

        if(response.HttpStatusCode!= System.Net.HttpStatusCode.OK)
        {
            return null;
        }

        var task = new Task()
        {
            Id = Guid.Parse(response.Item["id"].S),
            Description = response.Item["description"].S,
            Title = response.Item["title"].S
        };

        return task;
    }

    public async Task<Task[]> List()
    {
        var request = new ScanRequest()
        {
            TableName = _tableName
        };

        var response = await _amazonDynamoDB.ScanAsync(request);

        return response.Items.Select(item => new Task()
        {
            Id = Guid.Parse(item["id"].S),
            Description = item["description"].S,
            Title = item["title"].S
        }).ToArray();
    }
}
```

We are using the [DynamoDB low-level API](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/LowLevelDotNetItemsExample.html) because the `DynamoDBContext` has some issues when used with Native AOT. Now, go to the `PostTaskFunction` project and open the `Function.cs` file:

```csharp
public class Function
{
    private static TasksRepository _tasksRepository;

    static Function()
    {
        var tableName = Environment.GetEnvironmentVariable("TABLE_NAME");
        _tasksRepository = new TasksRepository(new AmazonDynamoDBClient(), tableName);
    }

    private static async System.Threading.Tasks.Task Main()
    {
        Func<APIGatewayHttpApiV2ProxyRequest, ILambdaContext, Task<APIGatewayHttpApiV2ProxyResponse>> handler = FunctionHandler;
        await LambdaBootstrapBuilder.Create(handler, new SourceGeneratorLambdaJsonSerializer<LambdaFunctionJsonSerializerContext>())
            .Build()
            .RunAsync();
    }

    public static async Task<APIGatewayHttpApiV2ProxyResponse> FunctionHandler(APIGatewayHttpApiV2ProxyRequest input, ILambdaContext context)
    {
        var request = JsonSerializer.Deserialize(input.Body, LambdaFunctionJsonSerializerContext.Default.RegisterTaskRequest)!;

        var task = new Task() { Id = Guid.NewGuid(), Description = request.Description, Title = request.Title };

        await _tasksRepository.Save(task);

        var body = JsonSerializer.Serialize(new RegisterTaskResponse(task.Id), LambdaFunctionJsonSerializerContext.Default.RegisterTaskResponse);

        return new APIGatewayHttpApiV2ProxyResponse
        {
            Body = body,
            StatusCode = 200,
            Headers = new Dictionary<string, string> { { "Content-Type", "application/json" } }
        };
    }
}

public record RegisterTaskRequest(string Description, string Title);

public record RegisterTaskResponse(Guid Id);

[JsonSerializable(typeof(RegisterTaskResponse))]
[JsonSerializable(typeof(RegisterTaskRequest))]
[JsonSerializable(typeof(Task))]
[JsonSerializable(typeof(APIGatewayHttpApiV2ProxyRequest))]
[JsonSerializable(typeof(APIGatewayHttpApiV2ProxyResponse))]
public partial class LambdaFunctionJsonSerializerContext : JsonSerializerContext
{
}
```

Just regular code generated by the `lambda.NativeAOT` template. Quite similar code is present under the `GetTaskFunction` and `ListTaskFunction` projects. Open the `template.yaml` file:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: >
  Sample SAM Template for aot-app

Globals:
  Function:
    Timeout: 10
    Handler: bootstrap
    Runtime: provided.al2
    MemorySize: 512
    Architectures:
      - x86_64
    Environment:
      Variables:
        TABLE_NAME: !Ref TasksTable

Resources:
  TasksTable:
    Type: AWS::Serverless::SimpleTable
    Properties:
      PrimaryKey:
        Name: id
        Type: String
      TableName: taskstable

  ListTaskFunction:
    Type: AWS::Serverless::Function
    Metadata:
      BuildMethod: dotnet7
    Properties:
      CodeUri: ./src/ListTaskFunction/
      Policies:
        - DynamoDBCrudPolicy:
            TableName: !Ref TasksTable
      Events:
        ListTask:
          Type: Api
          Properties:
            Path: /tasks
            Method: get

  PostTaskFunction:
    Type: AWS::Serverless::Function
    Metadata:
      BuildMethod: dotnet7
    Properties:
      CodeUri: ./src/PostTaskFunction/
      Policies:
        - DynamoDBCrudPolicy:
            TableName: !Ref TasksTable
      Events:
        ListTask:
          Type: Api
          Properties:
            Path: /tasks
            Method: post

  GetTaskFunction:
    Type: AWS::Serverless::Function
    Metadata:
      BuildMethod: dotnet7
    Properties:
      CodeUri: ./src/GetTaskFunction/
      Policies:
        - DynamoDBCrudPolicy:
            TableName: !Ref TasksTable
      Events:
        ListTask:
          Type: Api
          Properties:
            Path: /tasks/{id}
            Method: get

Outputs:
  TaskApi:
    Description: "API Gateway endpoint URL for Prod stage for Tasks function"
    Value: !Sub "https://${ServerlessRestApi}.execute-api.${AWS::Region}.amazonaws.com/Prod/tasks/"
```

*   `Transform`: For AWS SAM templates, you must include this section with a value of `AWS::Serverless-2016-10-31`
*   `Description`: A text string that describes the template.
*   `Globals`: Properties that are common to all your serverless functions, APIs, and simple tables.
*   `Resources`: The stack resources and their properties. Here we can use all the resources available in [Cloud Formation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html) plus: 
    *   `AWS::Serverless::Function`: Creates an AWS Lambda function, an AWS Identity and Access Management (IAM) execution role, and event source mappings that trigger the function.    
    *   `AWS::Serverless::SimpleTable`: Creates a DynamoDB table with a single attribute primary key. It is useful when data only needs to be accessed via a primary key.
    *   `AWS::Serverless::Api`: Creates a collection of Amazon API Gateway resources and methods that can be invoked through HTTPS endpoints.      
    *   `AWS::Serverless::HttpApi`: Creates an Amazon API Gateway HTTP API, which enables you to create RESTful APIs.        
    *   `AWS::Serverless::StateMachine`: Creates an AWS Step Functions state machine.      
    *   `AWS::Serverless::Application`: Embeds a serverless application from the [AWS Serverless Application](https://serverlessrepo.aws.amazon.com/applications) Repository or an Amazon S3 bucket as a nested application.      
    *   `AWS::Serverless::Connector`: Provides a simple and secure method of provisioning permissions between your serverless application resources.
    *   `AWS::Serverless::LayerVersion`: Creates a Lambda LayerVersion that contains library or runtime code needed by a Lambda Function.  
*   `Outputs`: The values that are returned whenever you view your stack's properties.
*   Other sections are `Parameters`, `Mappings`, `Conditions` and `Metadata`.
   
At the solution level, run `sam build` to build the serverless application and then `sam deploy --guided`, this command will guide you through the deployment. And that's it our application is up and running. You can check the official documentation [here](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-specification.html). Thanks, and happy coding.