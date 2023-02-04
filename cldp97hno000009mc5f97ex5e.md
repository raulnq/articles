# Using Amazon ElastiCache for Redis within a Lambda Function

In a previous [post](https://blog.raulnq.com/accessing-a-non-public-amazon-aurora-database-in-a-vpc-from-a-lambda-function), we reviewed how to access a non-public resource in a VPC from a Lambda function. Today we do the same but with a different Amazon resource, Redis:

> [Amazon Elasticache for Redis](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/WhatIs.html) is a distributed in-memory data store that is built on open-source [Redis](https://redis.io/docs/about/). It works with Redis APIs, and standard Redis client libraries, and uses open Redis data formats. Amazon ElastiCache for Redis can be used as a cache environment for your cloud applications in AWS.

## **Prerequisites**

* An Amazon ElastiCache Cluster (Redis).
    
* Install [**AWS CLI**](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-creds).
    
* Install the Amazon Lambda Templates (`dotnet new -i Amazon.Lambda.Templates`).
    
* Install the Amazon Lambda Tools (`dotnet tool install -g` [`Amazon.Lambda.Tools`](http://Amazon.Lambda.Tools)).
    
* Install [**AWS SAM CLI**](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html).
    

## Code

We will build an API to calculate a GUID value after a very expensive process. Redis will be used to store the value for 60 seconds using as a key a part of the request path. Run the following command to create the .NET projects and solution:

```bash
dotnet new lambda.EmptyFunction -n verySlowCalculationApi -o .
dotnet new sln -n aws-elasticache-sandbox
dotnet sln add --in-root src/verySlowCalculationApi
dotnet add src/verySlowCalculationApi package Amazon.Lambda.APIGatewayEvents
dotnet add src/verySlowCalculationApi package StackExchange.Redis
```

Open the solution and modify the `Function.cs` file with the following content:

```csharp
public class Function
{
    private IDatabase _database;

    public Function()
    {
        var connectionString = Environment.GetEnvironmentVariable("REDIS_CONFIGURATION");
        var connection = ConnectionMultiplexer.Connect(connectionString);
        _database = connection.GetDatabase();
    }

    public async Task<APIGatewayProxyResponse> FunctionHandler(APIGatewayProxyRequest input, ILambdaContext context)
    {
        var id = input.PathParameters["id"];

        var value = string.Empty;

        var cacheValue = await _database.StringGetAsync(id);

        if(cacheValue.HasValue)
        {
            value = cacheValue;
        }
        else
        {
            value = await VeryExpensiveCalculation();

            await _database.StringSetAsync(id, value, TimeSpan.FromSeconds(60), When.Always, CommandFlags.PreferMaster);
        }

        return new APIGatewayProxyResponse
        {
            Body = "{\"Value\":\""+ value + "\"}",
            StatusCode = 200,
            Headers = new Dictionary<string, string> { { "Content-Type", "application/json" } }
        };
    }

    public async Task<string> VeryExpensiveCalculation()
    {
        await Task.Delay(5000);

        return Guid.NewGuid().ToString();
    }
}
```

Create a `template.yml` file at the solution level:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: >
  Slow API

Parameters:
  Redis:
    Type: String
    Default: <REDIS_CONFIGURATION>
    Description: Redis configuration
  Vpc:
    Type: String
    Default: <REDIS_VPC>
    Description: Redis Vpc
  Subnet:
    Type: String
    Default: <REDIS_SUBNET>
    Description: Redis Subnet

Globals:
  Function:
    Timeout: 60
    MemorySize: 512
    Runtime: dotnet6
    Architectures:
      - x86_64
    Environment:
      Variables:
        REDIS_CONFIGURATION: !Ref Redis

Resources:
  LambdaSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: lambda security group
      VpcId: !Ref Vpc

  VerySlowFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: verySlowCalculationApi::verySlowCalculationApi.Function::FunctionHandler
      CodeUri: ./src/verySlowCalculationApi/
      VpcConfig:
        SecurityGroupIds:
          - !Ref LambdaSecurityGroup
        SubnetIds:
          - !Ref Subnet
      Policies:
        - AWSLambda_FullAccess
        - AWSLambdaVPCAccessExecutionRole
      Events:
        ListPosts:
          Type: Api
          Properties:
            Path: /{id}
            Method: get

Outputs:
  Api:
    Description: "API Gateway endpoint URL"
    Value: !Sub "https://${ServerlessRestApi}.execute-api.${AWS::Region}.amazonaws.com/Prod/{id}"
```

To connect the Lambda function to the VPC, we need to do the following:

* Assign the `AWSLambdaVPCAccessExecutionRole` and `AWSLambda_FullAccess` policies.
    
* Create the `LambdaSecurityGroup` security group(inside our VPC) and use it in our Lambda function(`SecurityGroupIds` property).
    
* Assign a subnet(the same subnet where our Redis cache lives) to the Lambda function(`SubnetIds` property)
    

Run `sam build` to build the application, and then `sam deploy --guided`, the command will guide you through the deployment. One thing to notice is that, by default, the Redis is not designed to be public. You can follow [this](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/accessing-elasticache.html) article to achieve it, but we recommend using a local instance to test your lambda function before deploying it. All the code is available [here](https://github.com/raulnq/aws-elasticache). Thanks, and happy coding.