---
title: "Lambda Powertools for .NET: Fetching Data from AWS Systems Manager Parameter Store and AWS Secrets Manager"
datePublished: Thu Jan 18 2024 13:59:36 GMT+0000 (Coordinated Universal Time)
cuid: clrja3apo000609i77g2zdqou
slug: lambda-powertools-for-net-fetching-data-from-aws-systems-manager-parameter-store-and-aws-secrets-manager
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1705505739818/ee8c169a-9946-48fe-b4b3-3c440f6e7a2f.png
tags: aws, net, aws-lambda, aws-secret-manager, aws-system-manager-parameter-store

---

The [external configuration storage](https://learn.microsoft.com/en-us/azure/architecture/patterns/external-configuration-store) pattern is widely used today. To support this, AWS provides a range of services for storing and managing configuration data, including:

* [AWS Systems Manager Parameter Store](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html) offers secure, scalable, centralized, and hierarchical storage for configuration data and secrets management. It can be used as a simple, low-cost solution for storing and managing configuration data and secrets when automatic secret rotation is not required.
    
* [AWS Secrets Manager](https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html) was designed specifically for confidential information. It provides a secure and scalable solution to store, retrieve, and rotate secrets. Keep in mind that Secrets Manager comes with a higher cost compared to Parameter Store.
    
* [AWS AppConfig](https://docs.aws.amazon.com/appconfig/latest/userguide/what-is-appconfig.html) is a service that enables us to manage and deploy configuration data. This service is ideal for situations where our configuration data updates and we need a rollback mechanism in case something goes wrong.
    

The [AWS Lambda Powertools Parameters](https://docs.powertools.aws.dev/lambda/dotnet/utilities/parameters/) utility simplifies working with the first two services by offering high-level functionality for retrieving configuration data. In this post, we will explore the features offered by the library while building AWS Lambda functions to retrieve values from AWS Systems Manager Parameter Store and AWS Secrets Manager.

## Pre-requisites

* Have an [IAM User](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html#id_users_create_console) with programmatic access.
    
* Install the Amazon Lambda Templates (`dotnet new -i Amazon.Lambda.Templates`)
    
* Install the Amazon Lambda Tools (`dotnet tool install -g`[`Amazon.Lambda.Tools`](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html#id_users_create_console))
    
* Install [AWS SAM CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html).
    

## Lambda Function

Run the following commands to set up our project:

```powershell
dotnet new lambda.EmptyFunction -n MyLambda -o .
dotnet add src/MyLambda package Amazon.Lambda.APIGatewayEvents
dotnet add src/MyLambda package AWS.Lambda.Powertools.Parameters
dotnet new sln -n PowerTools
dotnet sln add --in-root src/MyLambda
```

Open the solution and modify the `Function.cs` file as follows:

```csharp
using Amazon.Lambda.APIGatewayEvents;
using Amazon.Lambda.Core;
using System.Text.Json;
// Assembly attribute to enable the Lambda function's JSON input to be converted into a .NET class.
[assembly: LambdaSerializer(typeof(Amazon.Lambda.Serialization.SystemTextJson.DefaultLambdaJsonSerializer))]
namespace MyLambda;
public class Function
{
    public async Task<APIGatewayHttpApiV2ProxyResponse> FunctionHandler(APIGatewayHttpApiV2ProxyRequest input, ILambdaContext context)
    {
        var body = JsonSerializer.Serialize(new Response()
        {
        });

        return new APIGatewayHttpApiV2ProxyResponse
        {
            Body = body,
            StatusCode = 200,
            Headers = new Dictionary<string, string> { { "Content-Type", "application/json" } }
        };
    }

    public class Response
    {
    }
}
```

At the solution level, create a `template.yml` file as follows:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: >
  PowerTools Parameters

Resources:
  MyLambda:
    Type: AWS::Serverless::Function
    Properties:
      Timeout: 60
      MemorySize: 512
      Tracing: Active
      Runtime: dotnet6
      Architectures:
        - x86_64    
      Handler: MyLambda::MyLambda.Function::FunctionHandler
      CodeUri: ./src/MyLambda/
      Events:
        ListPosts:
          Type: Api
          Properties:
            Path: /api
            Method: get

Outputs:
  TaskApi:
    Description: "API Gateway endpoint URL"
    Value: 
      Fn::Sub:
        - https://${ApiId}.execute-api.${AWS::Region}.amazonaws.com/Prod/api
        - ApiId: 
            Ref: ServerlessRestApi
```

At the solution level, run the following commands to deploy the function:

```powershell
sam build
sam deploy --guided
```

## Parameter Store

### Retrieving a Single Parameter

Modify the `template.yml` file as follows:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: >
  PowerTools Parameters

Resources:
  BasicParameter:
    Type: AWS::SSM::Parameter
    Properties:
      Name: 'basicparameter'
      Type: String
      Value: 'Hello'
      Description: SSM Basic Parameter

  MyLambda:
    Type: AWS::Serverless::Function
    Properties:
      Timeout: 60
      MemorySize: 512
      Tracing: Active
      Runtime: dotnet6
      Architectures:
        - x86_64    
      Handler: MyLambda::MyLambda.Function::FunctionHandler
      CodeUri: ./src/MyLambda/
      Events:
        ListPosts:
          Type: Api
          Properties:
            Path: /api
            Method: get
      Policies:
        - SSMParameterReadPolicy:
            ParameterName:
              Ref: BasicParameter

Outputs:
  TaskApi:
    Description: "API Gateway endpoint URL"
    Value: 
      Fn::Sub:
        - https://${ApiId}.execute-api.${AWS::Region}.amazonaws.com/Prod/api
        - ApiId: 
            Ref: ServerlessRestApi
```

A new resource [`AWS::SSM::Parameter`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ssm-parameter.html) was added, and the corresponding permissions were granted to the function through the [SSMParameterReadPolicy](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-policy-template-list.html#ssm-parameter-read-policy) policy template. Open the `Function.cs` file and modify the `Function` class as follows:

```csharp
public class Function
{
    public async Task<APIGatewayHttpApiV2ProxyResponse> FunctionHandler(APIGatewayHttpApiV2ProxyRequest input, ILambdaContext context)
    {
        var ssmProvider = ParametersManager.SsmProvider;

        var value = await ssmProvider
            .GetAsync("basicparameter");

        var body = JsonSerializer.Serialize(new Response()
        {
            BasicParameterValue = value,
        });

        return new APIGatewayHttpApiV2ProxyResponse
        {
            Body = body,
            StatusCode = 200,
            Headers = new Dictionary<string, string> { { "Content-Type", "application/json" } }
        };
    }

    public class Response
    {
        public string? BasicParameterValue { get; set; }
    }
}
```

The `ParametersManager` static class contains a `SsmProvider` class that provides access to the `GetAsync` method for retrieving a single value.

### Retrieving a Multiple Parameters

We can use parameter hierarchies to help organize and manage parameters. A hierarchy consists of a parameter name that includes a path, defined by forward slashes. Modify the `template.yml` file as follows:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: >
  PowerTools Parameters

Resources:
  BasicParameter:
    Type: AWS::SSM::Parameter
    Properties:
      Name: 'basicparameter'
      Type: String
      Value: 'Hello'
      Description: SSM Basic Parameter

  MultipleParameter1:
    Type: AWS::SSM::Parameter
    Properties:
      Name: '/mylambda/multipleparameter1'
      Type: String
      Value: 'Parameter 1 Value'
      Description: SSM Multiple Parameter

  MultipleParameter2:
    Type: AWS::SSM::Parameter
    Properties:
      Name: '/mylambda/multipleparameter2'
      Type: String
      Value: 'Parameter 2 Value'
      Description: SSM Multiple Parameter

  MyLambda:
    Type: AWS::Serverless::Function
    Properties:
      Timeout: 60
      MemorySize: 512
      Tracing: Active
      Runtime: dotnet6
      Architectures:
        - x86_64    
      Handler: MyLambda::MyLambda.Function::FunctionHandler
      CodeUri: ./src/MyLambda/
      Events:
        ListPosts:
          Type: Api
          Properties:
            Path: /api
            Method: get
      Policies:
        - SSMParameterReadPolicy:
            ParameterName:
              Ref: BasicParameter
        - SSMParameterWithSlashPrefixReadPolicy:
            ParameterName: '/mylambda'

Outputs:
  TaskApi:
    Description: "API Gateway endpoint URL"
    Value: 
      Fn::Sub:
        - https://${ApiId}.execute-api.${AWS::Region}.amazonaws.com/Prod/api
        - ApiId: 
            Ref: ServerlessRestApi
```

In this case, permissions were granted using the [SSMParameterWithSlashPrefixReadPolicy](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-policy-template-list.html#ssm-parameter-slash-read-policy) policy template. Open the `Function.cs` file and modify the `Function` class as follows:

```csharp
public class Function
{
    public async Task<APIGatewayHttpApiV2ProxyResponse> FunctionHandler(APIGatewayHttpApiV2ProxyRequest input, ILambdaContext context)
    {
        var ssmProvider = ParametersManager.SsmProvider;

        var value = await ssmProvider
            .GetAsync("basicparameter");

        var values = await ssmProvider
            .Recursive()
            .GetMultipleAsync("/mylambda");

        var body = JsonSerializer.Serialize(new Response()
        {
            BasicParameterValue = value,
            MultipleParameterValues = values,
        });

        return new APIGatewayHttpApiV2ProxyResponse
        {
            Body = body,
            StatusCode = 200,
            Headers = new Dictionary<string, string> { { "Content-Type", "application/json" } }
        };
    }

    public class Response
    {
        public string? BasicParameterValue { get; set; }
        public IDictionary<string, string?>? MultipleParameterValues { get; set; }
    }
}
```

The `SsmProvider` class offers access to the `Recursive` method, which allows access to the `GetMultipleAsync` method for retrieving multiple values. Each retrieved item will consist of the parameter name and its corresponding value.

### Retrieving a Secure Parameter

The resource `AWS::SSM::Parameter` does not support the creation of SecureString parameters. In this case, we will use the following command to create the resource:

```powershell
aws ssm put-parameter --name "secureparameter" --type SecureString --value "ch4ng1ng-s3cr3t"
```

Modify the `template.yml` file as follows:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: >
  PowerTools Parameters

Resources:
  BasicParameter:
    Type: AWS::SSM::Parameter
    Properties:
      Name: 'basicparameter'
      Type: String
      Value: 'Hello'
      Description: SSM Basic Parameter

  MultipleParameter1:
    Type: AWS::SSM::Parameter
    Properties:
      Name: '/mylambda/multipleparameter1'
      Type: String
      Value: 'Parameter 1 Value'
      Description: SSM Multiple Parameter

  MultipleParameter2:
    Type: AWS::SSM::Parameter
    Properties:
      Name: '/mylambda/multipleparameter2'
      Type: String
      Value: 'Parameter 2 Value'
      Description: SSM Multiple Parameter

  MyLambda:
    Type: AWS::Serverless::Function
    Properties:
      Timeout: 60
      MemorySize: 512
      Tracing: Active
      Runtime: dotnet6
      Architectures:
        - x86_64    
      Handler: MyLambda::MyLambda.Function::FunctionHandler
      CodeUri: ./src/MyLambda/
      Events:
        ListPosts:
          Type: Api
          Properties:
            Path: /api
            Method: get
      Policies:
        - SSMParameterReadPolicy:
            ParameterName:
              Ref: BasicParameter
        - SSMParameterWithSlashPrefixReadPolicy:
            ParameterName: '/mylambda'
        - SSMParameterReadPolicy:
            ParameterName: 'secureparameter'

Outputs:
  TaskApi:
    Description: "API Gateway endpoint URL"
    Value: 
      Fn::Sub:
        - https://${ApiId}.execute-api.${AWS::Region}.amazonaws.com/Prod/api
        - ApiId: 
            Ref: ServerlessRestApi
```

The permissions were granted using the `SSMParameterReadPolicy` policy template. Open the `Function.cs` file and modify the `Function` class as follows:

```csharp
public class Function
{
    public async Task<APIGatewayHttpApiV2ProxyResponse> FunctionHandler(APIGatewayHttpApiV2ProxyRequest input, ILambdaContext context)
    {
        var ssmProvider = ParametersManager.SsmProvider;

        var value = await ssmProvider
            .GetAsync("basicparameter");

        var values = await ssmProvider
            .Recursive()
            .GetMultipleAsync("/mylambda");

        var secureValue = await ssmProvider
            .WithDecryption()
            .GetAsync("secureparameter");

        stopwatch.Stop();

        var body = JsonSerializer.Serialize(new Response()
        {
            BasicParameterValue = value,
            MultipleParameterValues = values,
            SecureParameterValue = secureValue,
        });

        return new APIGatewayHttpApiV2ProxyResponse
        {
            Body = body,
            StatusCode = 200,
            Headers = new Dictionary<string, string> { { "Content-Type", "application/json" } }
        };
    }

    public class Response
    {
        public string? BasicParameterValue { get; set; }
        public IDictionary<string, string?>? MultipleParameterValues { get; set; }
        public string? SecureParameterValue { get; set; }
    }
}
```

The `SsmProvider` class offers access to the `WithDecryption` method, which allows access to the `GetAsync` method for retrieving a single decrypted value.

## Secret Manager

Modify the `template.yml` file as follows:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: >
  PowerTools Parameters

Resources:
  BasicParameter:
    Type: AWS::SSM::Parameter
    Properties:
      Name: 'basicparameter'
      Type: String
      Value: 'Hello'
      Description: SSM Basic Parameter

  MultipleParameter1:
    Type: AWS::SSM::Parameter
    Properties:
      Name: '/mylambda/multipleparameter1'
      Type: String
      Value: 'Parameter 1 Value'
      Description: SSM Multiple Parameter

  MultipleParameter2:
    Type: AWS::SSM::Parameter
    Properties:
      Name: '/mylambda/multipleparameter2'
      Type: String
      Value: 'Parameter 2 Value'
      Description: SSM Multiple Parameter

  SecretParameter:
    Type: AWS::SecretsManager::Secret
    Properties:
      Description: Secrets Manager Secret
      Name: 'secret'
      GenerateSecretString:
        PasswordLength: 16

  MyLambda:
    Type: AWS::Serverless::Function
    Properties:
      Timeout: 60
      MemorySize: 512
      Tracing: Active
      Runtime: dotnet6
      Architectures:
        - x86_64    
      Handler: MyLambda::MyLambda.Function::FunctionHandler
      CodeUri: ./src/MyLambda/
      Events:
        ListPosts:
          Type: Api
          Properties:
            Path: /api
            Method: get
      Policies:
        - SSMParameterReadPolicy:
            ParameterName:
              Ref: BasicParameter
        - SSMParameterWithSlashPrefixReadPolicy:
            ParameterName: '/mylambda'
        - SSMParameterReadPolicy:
            ParameterName: 'secureparameter'
        - AWSSecretsManagerGetSecretValuePolicy:
            SecretArn: 
              Ref: SecretParameter

Outputs:
  TaskApi:
    Description: "API Gateway endpoint URL"
    Value: 
      Fn::Sub:
        - https://${ApiId}.execute-api.${AWS::Region}.amazonaws.com/Prod/api
        - ApiId: 
            Ref: ServerlessRestApi
```

A new resource [`AWS::SecretsManager::Secret`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-secretsmanager-secret.html) was added, and the corresponding permissions were granted to the function through the [AWSSecretsManagerGetSecretValuePolicy](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-policy-template-list.html#secrets-manager-get-secret-value-policy) policy template. Open the `Function.cs` file and modify the `Function` class as follows:

```csharp
public class Function
{
    public async Task<APIGatewayHttpApiV2ProxyResponse> FunctionHandler(APIGatewayHttpApiV2ProxyRequest input, ILambdaContext context)
    {
        var ssmProvider = ParametersManager.SsmProvider;

        var value = await ssmProvider
            .GetAsync("basicparameter");

        var values = await ssmProvider
            .Recursive()
            .GetMultipleAsync("/mylambda");

        var secureValue = await ssmProvider
            .WithDecryption()
            .GetAsync("secureparameter");

        var secretsProvider = ParametersManager.SecretsProvider;

        var secret = await secretsProvider
            .GetAsync("secret");

        var body = JsonSerializer.Serialize(new Response()
        {
            BasicParameterValue = value,
            MultipleParameterValues = values,
            SecureParameterValue = secureValue,
            Secret = secret
        });

        return new APIGatewayHttpApiV2ProxyResponse
        {
            Body = body,
            StatusCode = 200,
            Headers = new Dictionary<string, string> { { "Content-Type", "application/json" } }
        };
    }

    public class Response
    {
        public string? BasicParameterValue { get; set; }
        public IDictionary<string, string?>? MultipleParameterValues { get; set; }
        public string? SecureParameterValue { get; set; }
        public string? Secret { get; set; }
    }
}
```

The `ParametersManager` static class contains a `SecretsProvider` class that provides access to the `GetAsync` method for retrieving a single value.

## Transformations

Parameter values can be transformed using the `WithTransformation` method. The supported formats include Base64 and JSON. Modify the `template.yml` file as follows:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: >
  PowerTools Parameters

Resources:
  BasicParameter:
    Type: AWS::SSM::Parameter
    Properties:
      Name: 'basicparameter'
      Type: String
      Value: 'Hello'
      Description: SSM Basic Parameter

  MultipleParameter1:
    Type: AWS::SSM::Parameter
    Properties:
      Name: '/mylambda/multipleparameter1'
      Type: String
      Value: 'Parameter 1 Value'
      Description: SSM Multiple Parameter

  MultipleParameter2:
    Type: AWS::SSM::Parameter
    Properties:
      Name: '/mylambda/multipleparameter2'
      Type: String
      Value: 'Parameter 2 Value'
      Description: SSM Multiple Parameter

  SecretParameter:
    Type: AWS::SecretsManager::Secret
    Properties:
      Description: Secrets Manager Secret
      Name: 'secret'
      GenerateSecretString:
        PasswordLength: 16

  JsonParameter:
    Type: AWS::SSM::Parameter
    Properties:
      Name: 'jsonparameter'
      Type: String
      Value: "{\"Parameter1\":\"value1\", \"Parameter2\":\"value2\"}"
      Description: SSM Json Parameter

  MyLambda:
    Type: AWS::Serverless::Function
    Properties:
      Timeout: 60
      MemorySize: 512
      Tracing: Active
      Runtime: dotnet6
      Architectures:
        - x86_64    
      Handler: MyLambda::MyLambda.Function::FunctionHandler
      CodeUri: ./src/MyLambda/
      Events:
        ListPosts:
          Type: Api
          Properties:
            Path: /api
            Method: get
      Policies:
        - SSMParameterReadPolicy:
            ParameterName:
              Ref: BasicParameter
        - SSMParameterWithSlashPrefixReadPolicy:
            ParameterName: '/mylambda'
        - SSMParameterReadPolicy:
            ParameterName: 'secureparameter'
        - AWSSecretsManagerGetSecretValuePolicy:
            SecretArn: 
              Ref: SecretParameter
        - SSMParameterReadPolicy:
            ParameterName:
              Ref: JsonParameter

Outputs:
  TaskApi:
    Description: "API Gateway endpoint URL"
    Value: 
      Fn::Sub:
        - https://${ApiId}.execute-api.${AWS::Region}.amazonaws.com/Prod/api
        - ApiId: 
            Ref: ServerlessRestApi
```

A new resource [`AWS::SSM::Parameter`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-ssm-parameter.html) was added, but this time the value is a JSON object. In addition, the corresponding permissions were granted to the function. Open the `Function.cs` file and modify the `Function` class as follows:

```csharp
public class Function
{
    public async Task<APIGatewayHttpApiV2ProxyResponse> FunctionHandler(APIGatewayHttpApiV2ProxyRequest input, ILambdaContext context)
    {
        var ssmProvider = ParametersManager.SsmProvider;

        var value = await ssmProvider
            .GetAsync("basicparameter");

        var values = await ssmProvider
            .Recursive()
            .GetMultipleAsync("/mylambda");

        var secureValue = await ssmProvider
            .WithDecryption()
            .GetAsync("secureparameter");

        var secretsProvider = ParametersManager.SecretsProvider;

        var secret = await secretsProvider
            .GetAsync("secret");

        var configuration = await ssmProvider
            .WithTransformation(Transformation.Json)
            .GetAsync<Configuration>("jsonparameter");

        var body = JsonSerializer.Serialize(new Response()
        {
            BasicParameterValue = value,
            MultipleParameterValues = values,
            SecureParameterValue = secureValue,
            Secret = secret,
            Configuration = configuration
        });

        return new APIGatewayHttpApiV2ProxyResponse
        {
            Body = body,
            StatusCode = 200,
            Headers = new Dictionary<string, string> { { "Content-Type", "application/json" } }
        };
    }

    public class Response
    {
        public string? BasicParameterValue { get; set; }
        public IDictionary<string, string?>? MultipleParameterValues { get; set; }
        public string? SecureParameterValue { get; set; }
        public Configuration? Configuration { get; set; }
        public string? Secret { get; set; }
    }

    public record Configuration(string Parameter1, string Parameter2);
}
```

Deploy the application by running the commands `sam build` and `sam deploy`.

## Caching

By default, all parameters and their corresponding values are cached for five seconds. However, we can customize the duration using the `DefaultMaxAge` method.

```csharp
public class Function
{
    public async Task<APIGatewayHttpApiV2ProxyResponse> FunctionHandler(APIGatewayHttpApiV2ProxyRequest input, ILambdaContext context)
    {
        var stopwatch = new Stopwatch();
        stopwatch.Start();

        var ssmProvider = ParametersManager.SsmProvider;

        var value = await ssmProvider
            .DefaultMaxAge(TimeSpan.FromMinutes(1))
            .GetAsync("basicparameter");

        var values = await ssmProvider
            .Recursive()
            .GetMultipleAsync("/mylambda");

        var secureValue = await ssmProvider
            .WithDecryption()
            .GetAsync("secureparameter");

        var secretsProvider = ParametersManager.SecretsProvider;

        var secret = await secretsProvider
            .DefaultMaxAge(TimeSpan.FromMinutes(1))
            .GetAsync("secret");

        var configuration = await ssmProvider
            .WithTransformation(Transformation.Json)
            .GetAsync<Configuration>("jsonparameter");

        stopwatch.Stop();

        var body = JsonSerializer.Serialize(new Response()
        {
            BasicParameterValue = value,
            MultipleParameterValues = values,
            SecureParameterValue = secureValue,
            Secret = secret,
            Configuration = configuration,
            ElapsedMilliseconds = stopwatch.ElapsedMilliseconds
        });

        return new APIGatewayHttpApiV2ProxyResponse
        {
            Body = body,
            StatusCode = 200,
            Headers = new Dictionary<string, string> { { "Content-Type", "application/json" } }
        };
    }

    public class Response
    {
        public string? BasicParameterValue { get; set; }
        public IDictionary<string, string?>? MultipleParameterValues { get; set; }
        public string? SecureParameterValue { get; set; }
        public Configuration? Configuration { get; set; }
        public long ElapsedMilliseconds { get; set; }
        public string? Secret { get; set; }
    }

    public record Configuration(string Parameter1, string Parameter2);
}
```

We set the cache for both providers to 1 minute. Deploy the application by running the commands `sam build` and `sam deploy`.

## Disclaimer

During the [Lambda execution lifecycle](https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtime-environment.html), there are two phases where a parameter can be read:

* **Init phase**: This approach reduces API calls, as they are not made during every invocation. However, this can result in outdated parameters and potentially different values across concurrent execution environments.
    
* **During the invocation**: This approach keeps the value up to date but may result in higher retrieval costs and longer function durations due to the additional API call during every invocation.
    

Choose wisely according to your needs.

It's worth mentioning that, in addition to the Parameter Store and Secret Manager providers, the [DynamoDB](https://docs.powertools.aws.dev/lambda/dotnet/utilities/parameters/#dynamodb-provider) provider is also available. Furthermore, if that's not enough, it's quite easy to create our own [provider](https://docs.powertools.aws.dev/lambda/dotnet/utilities/parameters/#create-your-own-provider).

All the code can be found [here](https://github.com/raulnq/aws-powertools-parameters). Thanks, and happy coding.