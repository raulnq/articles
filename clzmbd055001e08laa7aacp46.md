---
title: "How to Throttle Requests in Amazon API Gateway"
datePublished: Fri Aug 09 2024 06:16:06 GMT+0000 (Coordinated Universal Time)
cuid: clzmbd055001e08laa7aacp46
slug: how-to-throttle-requests-in-amazon-api-gateway
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1722953297522/1ada7535-5923-4cc0-86a1-4291a0a8c4e4.png
tags: aws, throttling, aws-apigateway, amazon-api-gateway

---

Throttling requests in an API is a common practice for several reasons, mainly related to the following:

* **Performance**: Throttling safeguards backend services from being overwhelmed by excessive requests. Without it, a sudden traffic surge could overload servers, causing degraded performance or crashes.
    
* **Cost Management:** APIs often incur usage-based costs. Throttling helps manage expenses by limiting the number of processed requests.
    
* **Security**: Throttling restricts the number of requests a single client can make within a specific period. Enforcing rate limits prevents malicious users from flooding the API with excessive requests, ensuring service continuity for other users.
    
* **User Experience:** Throttling enhances the overall user experience by preventing system overload and maintaining API responsiveness. By managing the load effectively, we can provide more consistent performance for all users.
    

In Amazon API Gateway, throttling can be applied at multiple levels:

* **Account-level**: Sets limits that apply to all APIs within an AWS account in a specific region. The default rate limit is 10,000 requests per second, with a default burst limit of 5,000.
    
* **Stage-level**: Throttling can be applied at the stage level, affecting all requests to any method within that stage. Additionally, limits can be set at the method level within an API stage.
    
* **Usage plan-level**: Usage plans allow us to define limits for individual clients. Similar to stage-level, limits can also be set at the method level.
    

In this post, we will focus on the usage plan level. Throttling per client can be effectively managed using these components:

* **API keys**: Unique alphanumeric strings that identify a client of your API.
    
* **API stages**: Specific stage of our API.
    
* **Usage plans**: This component allows us to control access to our **API stages** by defining who can access them (using API keys) and how much they can use (using limits). An API key can be associated with multiple usage plans. A usage plan can be associated with multiple API stages. However, a given API key can only be linked to one usage plan per API stage.
    
    * **Throttling**:
        
        * **Burst limits:** The maximum number of requests allowed in a short period.
            
        * **Rate limits**: The steady-state rate of requests per second.
            
    * **Quota limits**: The maximum number of requests that can be made within a specified period (day, week, or month).
        

> Don't use API keys for authentication or authorization to control access to your APIs. If you have multiple APIs in a usage plan, a user with a valid API key for one API in that usage plan can access *all* APIs in that usage plan.

## Pre-requisites

* An [IAM User](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html#id_users_create_console) with programmatic access
    
* Install the [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-creds)
    
* Install the Amazon Lambda Templates (`dotnet new -i Amazon.Lambda.Templates`)
    
* Install the Amazon Lambda Tools (`dotnet tool install -g Amazon.Lambda.Tools`)
    
* Install [AWS SAM CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html)
    

## The Lambda function

Run the following commands to set up our Lambda function:

```powershell
dotnet new lambda.EmptyFunction -n MyLambda -o .
dotnet add src/MyLambda package Amazon.Lambda.APIGatewayEvents
dotnet new sln -n MyApplications
dotnet sln add --in-root src/MyLambda
```

Open the `Program.cs` file and update the content as follows:

```csharp
using Amazon.Lambda.APIGatewayEvents;
using Amazon.Lambda.Core;
using System.Text.Json;

[assembly: LambdaSerializer(typeof(Amazon.Lambda.Serialization.SystemTextJson.DefaultLambdaJsonSerializer))]

namespace MyLambda;

public class Function
{
    public APIGatewayHttpApiV2ProxyResponse FunctionHandler(APIGatewayHttpApiV2ProxyRequest input, ILambdaContext context)
    {
        context.Logger.LogInformation(JsonSerializer.Serialize(input));
        return new APIGatewayHttpApiV2ProxyResponse
        {
            Body = @"{""Message"":""Hello World""}",
            StatusCode = 200,
            Headers = new Dictionary<string, string> { { "Content-Type", "application/json" } }
        };
    }
}
```

## AWS SAM template

At the solution level, create a `template.yml` file with the following content:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: >
  SAM

Resources:
  MyApi:
    Type: AWS::Serverless::Api
    Properties:
      StageName: Prod

  MyApiFunction:
    Type: AWS::Serverless::Function
    Properties:
      Timeout: 60
      MemorySize: 512
      Tracing: Active
      Runtime: dotnet8
      Architectures:
        - x86_64    
      Handler: MyLambda::MyLambda.Function::FunctionHandler
      CodeUri: ./src/MyLambda/
      Events:
        Get:
          Type: Api
          Properties:
            RestApiId: !Ref MyApi
            Path: /hello-world
            Method: get

Outputs:
  MyApiEndpoint:
    Description: "API endpoint"
    Value: !Sub "https://${MyApi}.execute-api.${AWS::Region}.amazonaws.com/Prod/hello-world"
```

This file does not yet contain the usage plan configuration. Run the following commands to deploy the resources to AWS:

```powershell
sam build
sam deploy --guided
```

## Usage plan setup

Add the following resource to the `template.yml` file:

```yaml
  MyUsagePlan:
    Type: 'AWS::ApiGateway::UsagePlan'
    Properties: 
      UsagePlanName: MyUsagePlan
      ApiStages: 
        - ApiId: !Ref MyApi
          Stage: !Ref MyApi.Stage
      Throttle: 
        BurstLimit: 5
        RateLimit: 5
      Quota:
        Limit: 1000
        Period: DAY
```

The [`AWS::ApiGateway::UsagePlan`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-apigateway-usageplan.html) resource creates a usage plan and associates it with one or more API stages:

* [`ApiStages`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-apigateway-usageplan-apistage.html): The associated API stages of a usage plan.
    
* [`Quota`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-apigateway-usageplan-quotasettings.html): Maximum number of requests clients can make.
    
* [`Throttle`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-apigateway-usageplan-throttlesettings.html): Specifies the overall request rate (average per second) and burst capacity (number of requests that can be handled concurrently) when clients call the API.
    

## API key setup

Add the following resource to the `template.yml` file:

```yaml
  MyApiKey:
    Type: 'AWS::ApiGateway::ApiKey'
    Properties: 
      Enabled: true
      Name: MyApiKey

  MyUsagePlanKey:
    Type: 'AWS::ApiGateway::UsagePlanKey'
    Properties:
      KeyId: !Ref MyApiKey
      KeyType: "API_KEY"
      UsagePlanId: !Ref MyUsagePlan
```

The [`AWS::ApiGateway::ApiKey`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-apigateway-apikey.html) resource creates a unique key that we can distribute to our clients, and the [`AWS::ApiGateway::UsagePlanKey`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-apigateway-usageplankey.html) resource associates it with the usage plan.

## API Gateway setup

Update the `AWS::Serverless::Api` resource as follows:

```yaml
  MyApi:
    Type: AWS::Serverless::Api
    Properties:
      StageName: Prod
      Auth:
        ApiKeyRequired: 'true'
```

With this configuration, all methods for the API Gateway will require an API Key during invocation. Another option could be to put the configuration at the method level like this:

```yaml
  MyApiFunction:
    Type: AWS::Serverless::Function
    Properties:
      Timeout: 60
      MemorySize: 512
      Tracing: Active
      Runtime: dotnet8
      Architectures:
        - x86_64    
      Handler: MyLambda::MyLambda.Function::FunctionHandler
      CodeUri: ./src/MyLambda/
      Events:
        Get:
          Type: Api
          Properties:
            RestApiId: !Ref MyApi
            Path: /hello-world
            Method: get
            Auth:
              ApiKeyRequired: true
```

## Output setup

Add the following output to the `template.yml` file:

```yaml
  ApiKeyValue:
    Description: "CLI command to get the API key value"
    Value: !Sub "aws apigateway get-api-key --api-key ${MyApiKey.APIKeyId} --include-value --query \"value\" --output text"
```

The output above will show the command we can use to get the API Key value to call our endpoint. As a final step, redeploy the resources to AWS.

## **Testing**

We will use Postman to test our endpoint. If we try to access the endpoint without the `x-api-key` header, the result will look like this:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1723164889522/896f0515-8e34-426f-a31a-10fc1216aa7a.png align="center")

Run the AWS CLI command from the deployment output and use the result as the `x-api-key` header.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1723165133042/f0ae885d-b8d1-407d-b34b-012a9e31b418.png align="center")

In the current approach, we use CloudFormation resources to set up our usage plan. However, there is another way to do the same using only the `AWS::Serverless::Api` resource through the property [`ApiUsagePlan`](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-property-api-apiusageplan.html). Depending on your needs, this could be a shortcut for the configuration.

You can find the code and scripts [here](https://github.com/raulnq/aws-api-gateway-usage-plans). Thank you, and happy coding.