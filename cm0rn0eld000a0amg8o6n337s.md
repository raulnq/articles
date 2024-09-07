---
title: "How to Validate Requests in Amazon API Gateway"
datePublished: Sat Sep 07 2024 04:20:46 GMT+0000 (Coordinated Universal Time)
cuid: cm0rn0eld000a0amg8o6n337s
slug: how-to-validate-requests-in-amazon-api-gateway
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1725658067638/e8e3413a-8e5e-479f-95da-ea1e0e16fdb6.png
tags: aws, aws-apigateway, amazon-api-gateway

---

In the post, [Understanding Amazon API Gateway: Methods and Integrations](https://blog.raulnq.com/understanding-amazon-api-gateway-methods-and-integrations), we discussed how the method request section manages authorization, API KEY checks, and request validation. We've already covered authorization in articles like [How to Secure AWS Lambda Functions Using Amazon API Gateway and AWS IAM](https://blog.raulnq.com/how-to-secure-aws-lambda-functions-using-amazon-api-gateway-and-aws-iam) and [How to Use Lambda Authorizers to Validate Microsoft EntraID (Azure AD) Tokens in Amazon API Gateway](https://blog.raulnq.com/how-to-use-lambda-authorizers-to-validate-microsoft-entraid-azure-ad-tokens-in-amazon-api-gateway). For API KEY checks, see [How to Throttle Requests in Amazon API Gateway](https://blog.raulnq.com/how-to-throttle-requests-in-amazon-api-gateway). Today, we'll focus on the missing piece: [request validation](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-method-request-validation.html).

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
    public record RegisterPetRequest(string Name);
    public record RegisterPetResponse(Guid PetId, string Name);
    private readonly JsonSerializerOptions _options;

    public Function()
    {
        _options = new JsonSerializerOptions()
        {
            PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
        };
    }

    public APIGatewayHttpApiV2ProxyResponse FunctionHandler(APIGatewayHttpApiV2ProxyRequest input, ILambdaContext context)
    {
        var request = JsonSerializer.Deserialize<RegisterPetRequest>(input.Body, _options)!;
        var body = JsonSerializer.Serialize(new RegisterPetResponse(Guid.NewGuid(), request.Name), _options);
        return new APIGatewayHttpApiV2ProxyResponse
        {
            Body = body,
            StatusCode = 200,
            Headers = new Dictionary<string, string> { { "Content-Type", "application/json" } }
        };
    }
}
```

## **AWS SAM template**

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
      StageName: prod

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
        Post:
          Type: Api
          Properties:
            RestApiId: !Ref MyApi
            Path: /pets
            Method: post

Outputs:
  MyApiEndpoint:
    Description: "API endpoint"
    Value: !Sub "https://${MyApi}.execute-api.${AWS::Region}.amazonaws.com/prod/pets"
```

Run the following commands to deploy the resources to AWS:

```powershell
sam build
sam deploy --guided
```

## Body Validation

We can define a [JSON schema](https://json-schema.org/overview/what-is-jsonschema) as a [model](https://docs.aws.amazon.com/apigateway/latest/developerguide/models-mappings-models.html) that Amazon API Gateway will use to validate that the request body matches this schema. Update the `template.yml` file as follows:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: >
  SAM

Resources:
  MyApi:
    Type: AWS::Serverless::Api
    Properties:
      StageName: prod
      Models:
        RegisterPetRequest:
          type: "object"
          properties:
            name:
              type: "string"
              minLength: 1
          required:
            - name

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
        Post:
          Type: Api
          Properties:
            RestApiId: !Ref MyApi
            Path: /pets
            Method: post
            RequestModel:
              Model: RegisterPetRequest
              ValidateBody: true

Outputs:
  MyApiEndpoint:
    Description: "API endpoint"
    Value: !Sub "https://${MyApi}.execute-api.${AWS::Region}.amazonaws.com/prod/pets"
```

The `AWS::Serverless::Api` resource exposes the `Models` property, allowing us to define multiple schemas. Then, in the `AWS::Serverless::Function` under the event source `Api`, the `RequestModel` will configure the validation:

* `Model`: The name of the model defined earlier.
    
* `ValidateBody`: Enable body validation.
    
* `ValidateParameters`: Enable parameters (path, headers, and query string) validation.
    

## Header And Query String Validation

Amazon API Gateway can validate headers and query string parameters to ensure they are included in the request. Update the `AWS::Serverless::Function` resource as follows:

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
        Post:
          Type: Api
          Properties:
            RestApiId: !Ref MyApi
            Path: /pets
            Method: post
            RequestParameters:
             - method.request.header.traceid:
                 Required: true
             - method.request.querystring.version:
                 Required: true
            RequestModel:
              Model: RegisterPetRequest
              ValidateBody: true
              ValidateParameters: true
```

Within the `AWS::Serverless::Function` under the event source `Api`, the `RequestParameters` will partially set up the validation. We can define multiple request parameters in the following format:

* `method.request.querystring.parameter-name`: Represents query string parameters in the URL.
    
* `method.request.header.parameter-name`: Represents HTTP headers sent in the request.
    

Generally, `method.request` refers to the incoming request. It allows us to reference different parts of the HTTP request. To complete the configuration, `ValidateParameters` must be set to `true`.

> If we want to validate only headers and/or query strings, the `Model` property is still needed, but the `ValidateBody` property should be set to false.

## Gateway Responses

Amazon API Gateway provides a predefined set of [responses](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-gatewayResponse-definition.html) that automatically handle various errors. When validation fails, Amazon API Gateway returns a 400 status code, and a specific response will be returned depending on the error. In our case, we could expect a `BAD_REQUEST_PARAMETERS` or `BAD_REQUEST_BODY` response. These responses, by default, return a short descriptive error message but can be customized based on our needs. Update the `AWS::Serverless::Api` resource as follows:

```yaml
  MyApi:
    Type: AWS::Serverless::Api
    Properties:
      StageName: prod
      GatewayResponses:
        BAD_REQUEST_BODY:
          ResponseParameters:
            Headers:
              traceid: "method.request.header.traceid"
          ResponseTemplates:
            application/json: "{\"type\":\"https://example.net/validation-error\" ,\"detail\": \"$context.error.messageString\", \"title\":\"Validation error\" }"
      Models:
        RegisterPetRequest:
          type: "object"
          properties:
            name:
              type: "string"
              minLength: 1
          required:
            - name
```

In the `AWS::Serverless::Api` resource, the `GatewayResponses` property allows us to define multiple [response types](https://docs.aws.amazon.com/apigateway/latest/developerguide/supported-gateway-response-types.html) with the following structure:

* `ResponseParameters`: Allows us to specify headers that should be included in the response.
    
* `ResponseTemplates`: Allows us to define the body of the response.
    

`ResponseParameters` and `ResponseTemplates` let us define their content through a template that only supports simple variable substitutions. The template [can access](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-mapping-template-reference.html) `$context` variable values and `$stageVariables` property values, as well as method request parameters, in the form of `method.request.param-position.param-name`.

## Conclusions

Performing this kind of validation at the Amazon API Gateway level can eliminate unnecessary calls to the backend, thereby saving money on our bills. Additionally, this information enhances the [API definition file](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-export-api.html), making it easier to understand. Unfortunately, Amazon API Gateway models are defined using the [JSON Schema draft #4](http://json-schema.org/draft-04/schema#), which supports only the data types and validations available in this draft. You can find the final code [here](https://github.com/raulnq/aws-api-gateway-validations). Thanks, and happy coding.