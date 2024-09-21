---
title: "How to Use Non-Proxy Integration with Amazon API Gateway"
datePublished: Sat Sep 21 2024 21:08:59 GMT+0000 (Coordinated Universal Time)
cuid: cm1cn6vxe000a0ajz2cisc81b
slug: how-to-use-non-proxy-integration-with-amazon-api-gateway
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1726775840375/a50fcc77-e4b4-470e-987d-e65385b59c06.png
tags: aws, aws-lambda, api-gateway

---

As we discussed in our article [Understanding Amazon API Gateway: Methods and Integrations](https://blog.raulnq.com/understanding-amazon-api-gateway-methods-and-integrations), there are two types of integration with a backend service: proxy and non-proxy.

## Proxy Integration

In proxy integration, Amazon API Gateway forwards the entire request to the backend service, which processes the request and returns the response. There are two types of proxy integration: [**Lambda Proxy Integration**](https://docs.aws.amazon.com/apigateway/latest/developerguide/set-up-lambda-proxy-integrations.html) and [**HTTP Proxy Integration**](https://docs.aws.amazon.com/apigateway/latest/developerguide/setup-http-integrations.html#api-gateway-set-up-http-proxy-integration-on-proxy-resource). The main advantages are easy configuration and control over the request and response in the backend service. However, this approach limits our ability to manipulate the request and response from the backend service.

## Non-Proxy Integration

In non-proxy integration, Amazon API Gateway lets us control how requests are transformed before they are sent to the backend and how responses are returned. We explicitly configure the request and response mappings. There are two types: [Lambda Custom Integration](https://docs.aws.amazon.com/apigateway/latest/developerguide/set-up-lambda-custom-integrations.html) and [HTTP Custom Integration](https://docs.aws.amazon.com/apigateway/latest/developerguide/setup-http-integrations.html#set-up-http-custom-integrations). The main advantage is having control over the request and response structure, useful when integrating with backends that need specific formats. The downside is the increased complexity during configuration.

In this article, we focus on the last integration type, using a Lambda function as the backend service.

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
using Amazon.Lambda.Core;
using System.Text.Json;
// Assembly attribute to enable the Lambda function's JSON input to be converted into a .NET class.
[assembly: LambdaSerializer(typeof(Amazon.Lambda.Serialization.SystemTextJson.DefaultLambdaJsonSerializer))]
namespace MyLambda;
public class Function
{
    public record RegisterPetRequest(string Name, string ThrowError);
    public record RegisterPetResponse(Guid PetId, string Name);

    public RegisterPetResponse FunctionHandler(RegisterPetRequest input, ILambdaContext context)
    {
        if (!string.IsNullOrEmpty(input.ThrowError))
        {
            throw new ApplicationException("An error was thrown");
        }

        return new RegisterPetResponse(Guid.NewGuid(), input.Name);
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
    Type: AWS::ApiGateway::RestApi
    Properties:
      Description: API with lambda non proxy integration
      Name: apilambdanonproxy
      EndpointConfiguration:
        Types:
          - REGIONAL

  MyResource:
    Type: AWS::ApiGateway::Resource
    Properties:
      ParentId: !GetAtt MyApi.RootResourceId
      PathPart: 'pets'
      RestApiId: !Ref MyApi

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

  MyPermission:
    Type: AWS::Lambda::Permission
    DependsOn:
    - MyApi
    Properties:
      Action: lambda:InvokeFunction
      FunctionName: !Ref MyApiFunction
      Principal: apigateway.amazonaws.com

Outputs:
  MyApiEndpoint:
    Description: "API endpoint"
    Value: !Sub "https://${MyApi}.execute-api.${AWS::Region}.amazonaws.com/prod/pets"
  RestApiId:
    Description: "REST API id"
    Value: !Ref MyApi
```

* Usually, the [`AWS::Serverless::Function`](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-resource-function.html) resource simplifies setting up a Lambda function. However, in our case, it does not support non-proxy integration, so we will use CloudFormation resources to complete the setup.
    
* The [`AWS::ApiGateway::RestApi`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-apigateway-restapi.html) resource creates a REST API in Amazon API Gateway.
    
* The [`AWS::ApiGateway::Resource`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-apigateway-resource.html) resource creates a specific URI within our REST API, which can be associated with various HTTP methods.
    
* The [`AWS::Lambda::Permission`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-lambda-permission.html) resource grants permission for an AWS service or another AWS account to invoke a specific AWS Lambda function.
    

The [`AWS::ApiGateway::Method`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-apigateway-method.html) resource represents an HTTP method. This resource defines the settings for a particular HTTP method, such as the integration with the backend service. We will begin with the following setup:

```yaml
  MyMethod:
    Type: AWS::ApiGateway::Method
    Properties:
      ResourceId: !Ref MyResource
      RestApiId: !Ref MyApi
```

## Method Request and Method Response

From the method request perspective, we add the `AuthorizationType`, `HttpMethod`, and `RequestParameters` properties. For the method response, we define the `200` and `500` status codes and their corresponding response parameters within the `MethodResponses` property. Defining the `RequestParameters` and `MethodResponses` is necessary to set up and understand the integration section later.

```yaml
  MyMethod:
    Type: AWS::ApiGateway::Method
    Properties:
      ResourceId: !Ref MyResource
      RestApiId: !Ref MyApi
      AuthorizationType: NONE
      HttpMethod: POST
      RequestParameters:
        method.request.header.version: true
      MethodResponses:
        - StatusCode: '200'
          ResponseParameters:
            method.response.header.version: true
            method.response.header.id: true
        - StatusCode: '500'
          ResponseParameters:
            method.response.header.version: true
```

## Integration Request

The [Integration](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-apigateway-method-integration.html) property contains all the configuration for the backend service:

```yaml
  MyMethod:
    Type: AWS::ApiGateway::Method
    Properties:
      ResourceId: !Ref MyResource
      RestApiId: !Ref MyApi
      AuthorizationType: NONE
      HttpMethod: POST
      RequestParameters:
        method.request.header.version: true
      MethodResponses:
        - StatusCode: '200'
          ResponseParameters:
            method.response.header.version: true
            method.response.header.id: true
        - StatusCode: '500'
          ResponseParameters:
            method.response.header.version: true
      Integration:
        IntegrationHttpMethod: POST
        Type: AWS
        Uri: !Sub "arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${MyApiFunction.Arn}/invocations"
```

* The `IntegrationHttpMethod` property specifies the HTTP method used to invoke the backend service.
    
* Valid values for the `Type` property are `AWS`, `AWS_PROXY`, `HTTP`, `HTTP_PROXY`, and `MOCK`.
    
* For `HTTP` and `HTTP_PROXY` types, the `Uri` property holds the HTTP URL of our backend service. For `AWS` or `AWS_PROXY` types, the value follows the format `arn:aws:apigateway:{region}:{subdomain.service|service}:path|action/{service_api}`.
    

### Request Mapping

#### Body

To [map](https://docs.aws.amazon.com/apigateway/latest/developerguide/models-mappings.html) the body from the method request to the backend service, we use the `RequestTemplates` property. The property's value is a dictionary where the content type is the key and the template, written in [Velocity Template Language (VTL)](https://velocity.apache.org/engine/devel/vtl-reference.html), is the content. The following list shows the variables we can use during the mapping:

* The [`$input`](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-mapping-template-reference.html#input-variable-reference) variable lets us access the body, query strings, headers, and path variables.
    
* The [`$context`](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-mapping-template-reference.html#context-variable-reference) variable gives information about the Amazon API Gateway execution context.
    
* The [`$stageVariable`](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-mapping-template-reference.html#stagevariables-template-reference) can be used to inject environment values from the stage.
    
* The [`$util`](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-mapping-template-reference.html#util-template-reference) variable contains utility functions.
    

```yaml
  MyMethod:
    Type: AWS::ApiGateway::Method
    Properties:
      ResourceId: !Ref MyResource
      RestApiId: !Ref MyApi
      AuthorizationType: NONE
      HttpMethod: POST
      RequestParameters:
        method.request.header.version: true
      MethodResponses:
        - StatusCode: '200'
          ResponseParameters:
            method.response.header.version: true
            method.response.header.id: true
        - StatusCode: '500'
          ResponseParameters:
            method.response.header.version: true
      Integration:
        PassthroughBehavior: WHEN_NO_TEMPLATES
        IntegrationHttpMethod: POST
        Type: AWS
        Uri: !Sub "arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${MyApiFunction.Arn}/invocations"
        RequestTemplates:
          "application/json": "
            #set($inputRoot = $input.path('$'))
            {
              \"name\": \"$inputRoot.name\",
              \"throwError\":\"$input.params('throwerror')\"
            }"
```

The `PassthroughBehavior` property determines how the body of an unmapped content type will be sent to the backend service:

* `WHEN_NO_MATCH`: The request is passed to the backend without transformation when the content type does not match any content types associated with the mapping templates.
    
* `NEVER`: The request is rejected if no template matches.
    
* `WHEN_NO_TEMPLATES`: The request is passed to the backend without transformation only when no mapping template is defined in the integration request.
    

> We can use [https://velocity-template-tester.onrender.com/](https://velocity-template-tester.onrender.com/) to test our templates.

#### Headers

To map headers from the method request to the backend service, we use the `RequestParameters` property. The headers must also be defined in the method request.

```yaml
  MyMethod:
    Type: AWS::ApiGateway::Method
    Properties:
      ResourceId: !Ref MyResource
      RestApiId: !Ref MyApi
      AuthorizationType: NONE
      HttpMethod: POST
      RequestParameters:
        method.request.header.version: true
      MethodResponses:
        - StatusCode: '200'
          ResponseParameters:
            method.response.header.version: true
            method.response.header.id: true
        - StatusCode : '500'
          ResponseParameters:
            method.response.header.version: true
      Integration:
        PassthroughBehavior: WHEN_NO_TEMPLATES
        IntegrationHttpMethod: POST
        Type: AWS
        Uri: !Sub "arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${MyApiFunction.Arn}/invocations"
        RequestTemplates:
          "application/json": "
            #set($inputRoot = $input.path('$'))
            {
              \"name\": \"$inputRoot.name\",
              \"throwError\":\"$input.params('throwerror')\"
            }"
        RequestParameters:
          integration.request.header.version: method.request.header.version
```

The headers can come from the method request or static values. You can find more information [here](https://docs.aws.amazon.com/apigateway/latest/developerguide/request-response-data-mappings.html).

## Integration Response

Now, let's check the `IntegrationResponses` property, which specifies how to handle the response from the backend service. We can define multiple records under this property, each identified by the `StatusCode` property. These status codes must match the ones defined in the method response section. Every item has `ResponseTemplates` and `ResponseParameters` properties to map the body and headers, using the same rules we discussed earlier.

The `SelectionPattern` property helps us match an integration response set up with a specific response from the backend service. An empty value is considered the default mapping and will be evaluated last. When the backend service is an HTTP endpoint or AWS service, this pattern is evaluated against the response status code. An exception is the [Lambda functions](https://docs.aws.amazon.com/apigateway/latest/developerguide/handle-errors-in-lambda-integration.html), where error messages always appear in the `errorMessage` field in the response, so we match against that.

```yaml
  MyMethod:
    Type: AWS::ApiGateway::Method
    Properties:
      ResourceId: !Ref MyResource
      RestApiId: !Ref MyApi
      AuthorizationType: NONE
      HttpMethod: POST
      RequestParameters:
        method.request.header.version: true
      MethodResponses:
        - StatusCode: '200'
          ResponseParameters:
            method.response.header.version: true
            method.response.header.id: true
        - StatusCode : '500'
          ResponseParameters:
            method.response.header.version: true
      Integration:
        PassthroughBehavior: WHEN_NO_TEMPLATES
        IntegrationHttpMethod: POST
        Type: AWS
        Uri: !Sub "arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${MyApiFunction.Arn}/invocations"
        RequestTemplates:
          "application/json": "
            #set($inputRoot = $input.path('$'))
            {
              \"name\": \"$inputRoot.name\",
              \"throwError\":\"$input.params('throwerror')\"
            }"
        RequestParameters:
          integration.request.header.version: method.request.header.version
        IntegrationResponses:
        - StatusCode: '200'
          ResponseTemplates:
            "application/json": "
              #set($inputRoot = $input.path('$'))
              {
                \"petId\": \"$inputRoot.PetId\"
              }"
          ResponseParameters:
            method.response.header.version: "'v1'"
            method.response.header.id: integration.response.body.PetId
        - StatusCode: '500'
          SelectionPattern: ".*An error was thrown.*"
          ResponseTemplates:
            "application/json": "
              #set($inputRoot = $input.path('$'))
              {
                \"error\": \"$inputRoot.errorType\"
              }"
          ResponseParameters:
            method.response.header.version: "'v1'"
```

Run the following commands to deploy the resources to AWS:

```powershell
sam build
sam deploy --guided
aws apigateway create-deployment --rest-api-id [RestApiId] --stage-name prod --description "my stage" 
```

You can find the final code [here](https://github.com/raulnq/aws-lambda-non-proxy). Thanks, and happy coding.