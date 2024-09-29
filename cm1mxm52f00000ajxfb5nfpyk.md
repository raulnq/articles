---
title: "How to Use Non-Proxy Integration with Amazon API Gateway: OpenAPI Specification"
datePublished: Sun Sep 29 2024 01:58:28 GMT+0000 (Coordinated Universal Time)
cuid: cm1mxm52f00000ajxfb5nfpyk
slug: how-to-use-non-proxy-integration-with-amazon-api-gateway-openapi-specification
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1727463737486/bed00ba7-0cdb-46f0-bff6-c71e05136047.png
tags: aws, aws-lambda, openapi, aws-apigateway

---

In the article [How to Use Non-Proxy Integration with Amazon API Gateway](https://blog.raulnq.com/how-to-use-non-proxy-integration-with-amazon-api-gateway), we define resources such as `AWS::ApiGateway::RestApi`, `AWS::ApiGateway::Resource`, `AWS::Lambda::Permission`, and `AWS::ApiGateway::Method` in our AWS SAM template file instead of `AWS::Serverless::Function` and `AWS::Serverless::Api` due to their limitation in implementing the non-proxy integration. However, this method introduces a significant downside: redeploying the API to the corresponding stage must be done using an AWS CLI command. This necessity arises from the lack of this feature in CloudFormation resources, at least for Rest APIs. Fortunately, this feature is already included when using `AWS::Serverless::Api` resources.

So, in this article, we will explore an alternative to continue using only serverless resources. As we discuss in [Linking an OpenAPI Specification to AWS Lambda Functions using AWS SAM](https://blog.raulnq.com/linking-an-openapi-specification-to-aws-lambda-functions-using-aws-sam), an [OpenAPI specification](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-import-api.html) can define an API. Run the following command:

```powershell
aws apigateway get-export --parameters extensions='apigateway' --rest-api-id <rest_api_id>--stage-name prod --export-type oas30 openapi_with_extensions.yaml --accepts application/yaml
```

The resulting file will be the starting point for creating a file like this:

```yaml
openapi: "3.0.1"
info:
  title: "apilambdanonproxy"
  description: "API with lambda non proxy integration"
paths:
  /pets:
    post:
      parameters:
      - name: "version"
        in: "header"
        required: true
        schema:
          type: "string"
      responses:
        "500":
          description: "500 response"
          headers:
            version:
              schema:
                type: "string"
          content: {}
        "200":
          description: "200 response"
          headers:
            id:
              schema:
                type: "string"
            version:
              schema:
                type: "string"
          content: {}
      x-amazon-apigateway-integration:
        httpMethod: "POST"
        uri: 
          Fn::Sub: "arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${MyApiFunction.Arn}/invocations"
        responses:
          ".*An error was thrown.*":
            statusCode: "500"
            responseParameters:
              method.response.header.version: "'v1'"
            responseTemplates:
              application/json: " #set($inputRoot = $input.path('$')) { \"error\"\
                : \"$inputRoot.errorType\" }"
          default:
            statusCode: "200"
            responseParameters:
              method.response.header.id: "integration.response.body.PetId"
              method.response.header.version: "'v1'"
            responseTemplates:
              application/json: " #set($inputRoot = $input.path('$')) { \"petId\"\
                : \"$inputRoot.PetId\" }"
        requestParameters:
          integration.request.header.version: "method.request.header.version"
        requestTemplates:
          application/json: " #set($inputRoot = $input.path('$')) { \"name\": \"$inputRoot.name\"\
            , \"throwError\":\"$input.params('throwerror')\" }"
        passthroughBehavior: "when_no_templates"
        type: "aws"
components: {}
```

As we can see, the section under the property `x-amazon-apigateway-integration` contains all the integration setup and can easily match what we did. For more information about the OpenAPI extensions for Amazon API Gateway, click [here](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-swagger-extensions.html). Update the `template.yml` file as follows:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: >
  SAM

Resources:
  MyApi:
    Type: AWS::Serverless::Api
    Properties:
     OpenApiVersion: '3.0.1'
     DefinitionBody:
       'Fn::Transform':
         Name: 'AWS::Include'
         Parameters:
           Location: openapi.yaml
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
  RestApiId:
    Description: "REST API id"
    Value: !Ref MyApi
```

Run the following commands to deploy the resources to AWS:

```powershell
sam build
sam deploy --guided
```

As a result, we have the same API but using serverless resources only. You can find the final code [here](https://github.com/raulnq/aws-lambda-non-proxy/tree/openapi). Thanks, and happy coding.