---
title: "Using RDS Data API with AWS Lambda Functions and Amazon Aurora Serverless v2"
datePublished: Wed Feb 28 2024 22:55:59 GMT+0000 (Coordinated Universal Time)
cuid: clt6eb09p000109jrdpqe5ebi
slug: using-rds-data-api-with-aws-lambda-functions-and-amazon-aurora-serverless-v2
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1709135161734/29549fe9-c420-4932-a1ea-df32590f8da4.png
tags: net, rds, aws-lambda, aws-aurora-serverless-v2, rds-data-api

---

The [RDS Data API](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/data-api.html#data-api.regions) is a service that allows us to interact with databases hosted on Amazon RDS using HTTP REST API calls. It provides a secure and easy-to-use interface for executing SQL queries, managing transactions, and fetching results without requiring direct access to the underlying database instances.

RDS Data API offers several advantages and disadvantages, which are essential to consider based on our specific use case and requirements. Here's a summary of the pros and cons:

**Pros:**

* **Simplified Access:** Provides a simple HTTP-based interface for interacting with our databases, eliminating the need for managing connections in our application code.
    
* **Security**: Integration with AWS IAM and AWS Secrets Manager for authentication and authorization.
    
* **Scalability:** Handles database connections and connection pooling, enabling our applications to scale efficiently without being limited by the number of available connections.
    
* **Cross-Region Access:** We can access our databases from anywhere using the RDS Data API, making it suitable for globally distributed applications.
    

**Cons:**

* **Latency**: Since the RDS Data API is an HTTP-based interface, there may be additional latency compared to direct database connections.
    
* **Limited Functionality:** RDS Data API may not support all database features or operations available through database connections.
    
* **Vendor Lock-In:** Using the RDS Data API may introduce vendor lock-in to the AWS ecosystem.
    
* **Cost**: The first 1 million requests each month are free. However, there are charges for additional requests and data transfer. Also, using AWS Secrets Manager and AWS CloudTrail (if activated) will result in extra costs.
    

While the RDS Data API provides several benefits for working with RDS databases, there are situations where it might not be the ideal choice, especially if our application requires high throughput, large data transfers, and/or bulk operations. On the other hand, serverless applications that don't need the features mentioned earlier could be a perfect fit. With the [announcement](https://aws.amazon.com/es/blogs/database/introducing-the-data-api-for-amazon-aurora-serverless-v2-and-amazon-aurora-provisioned-clusters/) of the Data API for Amazon Aurora Serverless v2, new possibilities have opened up for our AWS Lambda function applications.

In the rest of this article, we will write an AWS Lambda function to demonstrate how to use the AWS SDK for .NET to interact with the RDS Data API.

## **Pre-requisites**

* Have an [IAM User](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html#id_users_create_console) with programmatic access.
    
* Install the [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-creds).
    
* Install the Amazon Lambda Templates (`dotnet new -i Amazon.Lambda.Templates`)
    
* Install the Amazon Lambda Tools (`dotnet tool install -g Amazon.Lambda.Tools`)
    
* Install [AWS SAM CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html).
    

## The Database

In a previous article, we saw [how to create an Amazon Aurora serverless v2 using AWS SAM](https://blog.raulnq.com/creating-an-amazon-aurora-serverless-v2-database-using-aws-sam). We will use the script presented there as a starting point. Create a `template.yml` file with the following content:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: >
  SAM Template

Resources:
  DBSubnetGroup:
    Type: AWS::RDS::DBSubnetGroup
    Properties:
      DBSubnetGroupDescription: Aurora DB subnet group
      SubnetIds:
        - <MY_SUBNET_1>
        - <MY_SUBNET_2>

  DBSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupName: Aurora DB SG
      GroupDescription: Ingress rules for Aurora DB
      VpcId: <MY_VPC>
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 5432
          ToPort: 5432
          CidrIp: 0.0.0.0/0

  DBCluster:
    Type: AWS::RDS::DBCluster
    DeletionPolicy: Delete
    Properties:
      DatabaseName: mydatabase
      DBClusterIdentifier: my-dbcluster
      DBSubnetGroupName: !Ref DBSubnetGroup
      Engine: aurora-postgresql
      EngineVersion: 15.4
      MasterUsername: <MY_USER>
      ManageMasterUserPassword: True
      Port: 5432
      EnableHttpEndpoint: true
      ServerlessV2ScalingConfiguration:
        MaxCapacity: 1.0
        MinCapacity: 0.5
      VpcSecurityGroupIds:
        - !Ref DBSecurityGroup

  DBInstance:
    Type: AWS::RDS::DBInstance
    Properties:
      DBClusterIdentifier: !Ref DBCluster
      DBInstanceIdentifier: my-dbinstance
      DBInstanceClass: db.serverless
      Engine: aurora-postgresql

Outputs:
  DBSecret:
    Description: Secret arn
    Value: !GetAtt DBCluster.MasterUserSecret.SecretArn
  DBCluster:
    Description: Cluster arn
    Value: !GetAtt DBCluster.DBClusterArn
```

In this script, at the cluster level, we set `ManageMasterUserPassword` to `true` to manage the password using [AWS Secrets Manager](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-secrets-manager.html). We also enable the RDS Data API by setting the property `EnableHttpEndpoint` to `true`. We are adding two outputs to display the ARNs for the cluster and the secret. Run the following commands to create the database:

```powershell
sam build
sam deploy --guided
```

Let's create a table using the AWS CLI by running the following commands:

```powershell
aws rds-data execute-statement --resource-arn "<MY_CLUSTER_ARN>" --database "mydatabase" --secret-arn "<MY_SECRET_ARN>" --sql "CREATE TABLE Tasks (Id VARCHAR(255) PRIMARY KEY, Name VARCHAR(50), Description VARCHAR(255));"
```

## The Lambda Function

Run the following commands to set up our project:

```powershell
dotnet new lambda.EmptyFunction -n MyApi-o .
dotnet add src/MyApi package Amazon.Lambda.APIGatewayEvents
dotnet add src/MyApi package AWSSDK.RDSDataService
dotnet new sln -n MyApi
dotnet sln add --in-root src/MyApi
```

Open the solution and modify the `Function.cs` file as follows:

```csharp
using Amazon.Lambda.APIGatewayEvents;
using Amazon.Lambda.Core;
using Amazon.RDSDataService;
using Amazon.RDSDataService.Model;
using System.Text.Json;
// Assembly attribute to enable the Lambda function's JSON input to be converted into a .NET class.
[assembly: LambdaSerializer(typeof(Amazon.Lambda.Serialization.SystemTextJson.DefaultLambdaJsonSerializer))]
namespace MyApi;

public class Function
{
    private readonly IAmazonRDSDataService _client;
    private readonly string _secretArn;
    private readonly string _clusterArn;
    private readonly string _database;

    public Function()
    {
        _client = new AmazonRDSDataServiceClient();
        _secretArn = Environment.GetEnvironmentVariable("SECRET_ARN")!;
        _clusterArn = Environment.GetEnvironmentVariable("CLUSTER_ARN")!;
        _database = Environment.GetEnvironmentVariable("DATABASE")!;
    }

    public async Task<APIGatewayHttpApiV2ProxyResponse> Register(APIGatewayHttpApiV2ProxyRequest input, ILambdaContext context)
    {
        var registerTaskrequest = JsonSerializer.Deserialize<RegisterTaskRequest>(input.Body)!;
        var taskId = Guid.NewGuid().ToString();
        var request = new ExecuteStatementRequest()
        {
            Sql = "insert into tasks(Id, Name, Description) VALUES(:id, :name, :description)",
            ResourceArn = _clusterArn,
            SecretArn = _secretArn,
            Parameters = new List<SqlParameter>()
            {
                new SqlParameter(){ Name = "id", Value = new Field(){ StringValue=taskId } },
                new SqlParameter(){ Name = "name", Value = new Field(){ StringValue=registerTaskrequest.Name } },
                new SqlParameter(){ Name = "description", Value = new Field(){ StringValue=registerTaskrequest.Description } }
            },
            Database = _database,
        };

        try
        {
            var response = await _client.ExecuteStatementAsync(request);
            var registerTaskResponse = JsonSerializer.Serialize(new RegisterTaskResponse(taskId));
            return new APIGatewayHttpApiV2ProxyResponse
            {
                Body = registerTaskResponse,
                StatusCode = 200,
                Headers = new Dictionary<string, string> { { "Content-Type", "application/json" } }
            };
        }
        catch (DatabaseErrorException)
        {
            return new APIGatewayHttpApiV2ProxyResponse
            {
                StatusCode = 500,
                Headers = new Dictionary<string, string> { { "Content-Type", "application/json" } }
            };
        }
    }

    public async Task<APIGatewayHttpApiV2ProxyResponse> Get(APIGatewayHttpApiV2ProxyRequest input, ILambdaContext context)
    {
        var taskId = input.PathParameters["taskId"];
        var request = new ExecuteStatementRequest()
        {
            Sql = "select Id, Name, Description from tasks where id = :id",
            ResourceArn = _clusterArn,
            SecretArn = _secretArn,
            Parameters = new List<SqlParameter>()
            {
                new SqlParameter(){ Name = "id", Value = new Field(){ StringValue=taskId } },
            },
            Database = _database,
            FormatRecordsAs =  RecordsFormatType.JSON
        };

        try
        {
            var response = await _client.ExecuteStatementAsync(request);
            var tasks = JsonSerializer.Deserialize<@Task[]>(response.FormattedRecords, new JsonSerializerOptions() { PropertyNameCaseInsensitive = true })!;
            if(!tasks.Any())
            {
                return new APIGatewayHttpApiV2ProxyResponse
                {
                    StatusCode = 404,
                    Headers = new Dictionary<string, string> { { "Content-Type", "application/json" } }
                };
            }

            var task = JsonSerializer.Serialize(tasks.First());
            return new APIGatewayHttpApiV2ProxyResponse
            {
                Body = task,
                StatusCode = 200,
                Headers = new Dictionary<string, string> { { "Content-Type", "application/json" } }
            };
        }
        catch (DatabaseErrorException)
        {
            return new APIGatewayHttpApiV2ProxyResponse
            {
                StatusCode = 500,
                Headers = new Dictionary<string, string> { { "Content-Type", "application/json" } }
            };
        }
    }

    public async Task<APIGatewayHttpApiV2ProxyResponse> List(APIGatewayHttpApiV2ProxyRequest input, ILambdaContext context)
    {
        var request = new ExecuteStatementRequest()
        {
            Sql = "select Id, Name, Description from tasks",
            ResourceArn = _clusterArn,
            SecretArn = _secretArn,
            Database = _database,
            FormatRecordsAs = RecordsFormatType.JSON
        };

        try
        {
            var response = await _client.ExecuteStatementAsync(request);
            var tasks = JsonSerializer.Deserialize<@Task[]>(response.FormattedRecords, new JsonSerializerOptions() { PropertyNameCaseInsensitive = true })!;
            return new APIGatewayHttpApiV2ProxyResponse
            {
                Body = JsonSerializer.Serialize(tasks),
                StatusCode = 200,
                Headers = new Dictionary<string, string> { { "Content-Type", "application/json" } }
            };
        }
        catch (DatabaseErrorException)
        {
            return new APIGatewayHttpApiV2ProxyResponse
            {
                StatusCode = 500,
                Headers = new Dictionary<string, string> { { "Content-Type", "application/json" } }
            };
        }
    }
}

public record RegisterTaskRequest(string Name, string Description);
public record RegisterTaskResponse(string Id);
public record @Task(string Id, string Name, string Description);
```

We define functions to register, retrieve, and list all the task records from the database. In all cases, we use the `ExecuteStatementAsync` method, but the complete list includes:

* `ExecuteStatementAsync`: Runs a SQL statement on a database.
    
* `BatchExecuteStatementAsync`: Runs a batch SQL statement over an array of data for bulk update and insert operations.
    
* `BeginTransactionAsync`: Starts a SQL transaction.
    
* `CommitTransactionAsync`: Ends a SQL transaction and commits the changes.
    
* `RollbackTransactionAsync`: Performs a rollback of a transaction.
    

All these operations share the following properties:

* `ResourceArn`: The ARN of the Aurora DB cluster.
    
* `SecretArn`: The name or ARN of the secret that enables access to the DB cluster.
    
* `Database`: The name of the database.
    

When using the `ExecuteStatementAsync` operation, we can get the query results in JSON format by setting the `FormatRecordsAs` property. Let's update the `template.yml` to include the definition of the AWS Lambda functions:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: >
  SAM Template

Globals:
  Function:
    Runtime: dotnet6
    Timeout: 30
    MemorySize: 512
    Architectures:
      - x86_64
    Environment:
      Variables:
        SECRET_ARN: !GetAtt DBCluster.MasterUserSecret.SecretArn
        CLUSTER_ARN: !GetAtt DBCluster.DBClusterArn
        DATABASE: mydatabase

Resources:
  DBSubnetGroup:
    Type: AWS::RDS::DBSubnetGroup
    Properties:
      DBSubnetGroupDescription: Aurora DB subnet group
      SubnetIds:
        - <MY_SUBNET_1>
        - <MY_SUBNET_2>

  DBSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupName: Aurora DB SG
      GroupDescription: Ingress rules for Aurora DB
      VpcId: <MY_VPC>
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 5432
          ToPort: 5432
          CidrIp: 0.0.0.0/0

  DBCluster:
    Type: AWS::RDS::DBCluster
    DeletionPolicy: Delete
    Properties:
      DatabaseName: mydatabase
      DBClusterIdentifier: my-dbcluster
      DBSubnetGroupName: !Ref DBSubnetGroup
      Engine: aurora-postgresql
      EngineVersion: 15.4
      MasterUsername: <MY_USER>
      ManageMasterUserPassword: True
      Port: 5432
      EnableHttpEndpoint: true
      ServerlessV2ScalingConfiguration:
        MaxCapacity: 1.0
        MinCapacity: 0.5
      VpcSecurityGroupIds:
        - !Ref DBSecurityGroup

  DBInstance:
    Type: AWS::RDS::DBInstance
    Properties:
      DBClusterIdentifier: !Ref DBCluster
      DBInstanceIdentifier: my-dbinstance
      DBInstanceClass: db.serverless
      Engine: aurora-postgresql

  RegisterFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: MyApi::MyApi.Function::Register
      CodeUri: ./src/MyApi/
      Policies:
        - Statement:
          - Effect: Allow
            Action: 
              - rds-data:BatchExecuteStatement
              - rds-data:BeginTransaction
              - rds-data:CommitTransaction   
              - rds-data:RollbackTransaction
              - rds-data:ExecuteStatement
            Resource: !GetAtt DBCluster.DBClusterArn
          - Effect: Allow
            Action: 
              - secretsmanager:GetSecretValue
            Resource: !GetAtt DBCluster.MasterUserSecret.SecretArn
      Events:
        RegisterTask:
          Type: Api
          Properties:
            Path: /tasks
            Method: post

  GetFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: MyApi::MyApi.Function::Get
      CodeUri: ./src/MyApi/
      Policies:
        - Statement:
          - Effect: Allow
            Action: 
              - rds-data:BatchExecuteStatement
              - rds-data:BeginTransaction
              - rds-data:CommitTransaction   
              - rds-data:RollbackTransaction
              - rds-data:ExecuteStatement
            Resource: !GetAtt DBCluster.DBClusterArn
          - Effect: Allow
            Action: 
              - secretsmanager:GetSecretValue
            Resource: !GetAtt DBCluster.MasterUserSecret.SecretArn
      Events:
        ListTask:
          Type: Api
          Properties:
            Path: /tasks/{taskId}
            Method: get

  ListFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: MyApi::MyApi.Function::List
      CodeUri: ./src/MyApi/
      Policies:
        - Statement:
          - Effect: Allow
            Action: 
              - rds-data:BatchExecuteStatement
              - rds-data:BeginTransaction
              - rds-data:CommitTransaction   
              - rds-data:RollbackTransaction
              - rds-data:ExecuteStatement
            Resource: !GetAtt DBCluster.DBClusterArn
          - Effect: Allow
            Action: 
              - secretsmanager:GetSecretValue
            Resource: !GetAtt DBCluster.MasterUserSecret.SecretArn
      Events:
        ListTask:
          Type: Api
          Properties:
            Path: /tasks
            Method: get

Outputs:
  DBSecret:
    Description: Secret arn
    Value: !GetAtt DBCluster.MasterUserSecret.SecretArn
  DBCluster:
    Description: Cluster arn
    Value: !GetAtt DBCluster.DBClusterArn
  Api:
    Description: "API Gateway endpoint URL"
    Value: 
      Fn::Sub: "https://${ServerlessRestApi}.execute-api.${AWS::Region}.amazonaws.com/Prod/tasks/"
```

Here, we define three AWS Lambda functions, each with the necessary permissions to access the RDS Data API operations mentioned earlier and to read the secret that contains the database credentials. Run the following commands to deploy the AWS Lambda functions:

```powershell
sam build
sam deploy --guided
```

Copy the output URL and try out the functions. Integrating the RDS Data API with AWS Lambda functions is a good way to avoid dealing with connections or VPC restrictions. However, it does come with the downside of increased latency. Therefore, carefully consider if it's the right choice for your needs. All the code can be found [here](https://github.com/raulnq/aurora-data-api). Thanks, and happy coding.