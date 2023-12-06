---
title: "Linking an OpenAPI Specification to AWS Lambda Functions using AWS SAM"
datePublished: Wed Dec 06 2023 23:00:16 GMT+0000 (Coordinated Universal Time)
cuid: clpudgypn000008l91egpeyln
slug: linking-an-openapi-specification-to-aws-lambda-functions-using-aws-sam
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1701821524170/e98d31f4-ceec-416a-936a-ed5493a71f33.png
tags: aws, net, aws-lambda, openapi, aws-apigateway

---

> The OpenAPI Specification (OAS) defines a standard, language-agnostic interface to HTTP APIs which allows both humans and computers to discover and understand the capabilities of the service without access to source code, documentation, or through network traffic inspection. When properly defined, a consumer can understand and interact with the remote service with a minimal amount of implementation logic.

In the .NET world, documenting our APIs using the [OpenAPI specification](https://swagger.io/specification/) is a relatively simple task, as demonstrated in the following [link](https://learn.microsoft.com/en-us/aspnet/core/tutorials/web-api-help-pages-using-swagger). How can we accomplish the same when working with AWS Lambda functions? In this article, we will attempt to answer that question, although with a little extra effort.

## **Pre-requisites**

* Install the [**AWS CLI**](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-creds).
    
* Install the [**AWS SAM CLI**](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html).
    

## The Sample Code

Download the sample code [here](https://github.com/raulnq/aws-lambda-openapi/tree/main). Open the solution and navigate to the `template.yaml` file:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: >
  SAM Template

Globals:
  Function:
    Timeout: 30
    MemorySize: 512
    Architectures:
      - x86_64
    Environment:
      Variables:
        TABLE_NAME:
          Ref: DynamoTable
Resources:
  DynamoTable:
    Type: AWS::Serverless::SimpleTable
    Properties:
      PrimaryKey:
        Name: petId
        Type: String
      TableName: petstable
  ApiGatewayApi:
    Type: AWS::Serverless::Api
    Properties:
      Name: petsapi
      StageName: Prod
      Cors:
        AllowMethods: "'*'"
        AllowHeaders: "'*'"
        AllowOrigin: "'*'"
  RegisterFunction:
    Type: AWS::Serverless::Function
    Properties:
      Runtime: dotnet6
      Handler: PetsApi::PetsApi.Function::Register
      CodeUri: ./src/PetsApi/
      Policies:
        - DynamoDBCrudPolicy:
            TableName:
              Ref: DynamoTable
      Events:
        RegisterPet:
          Type: Api
          Properties:
            Path: /pets
            Method: post
            RestApiId:
              Ref: ApiGatewayApi
  GetFunction:
    Type: AWS::Serverless::Function
    Properties:
      Runtime: dotnet6
      Handler: PetsApi::PetsApi.Function::Get
      CodeUri: ./src/PetsApi/
      Policies:
        - DynamoDBCrudPolicy:
            TableName:
              Ref: DynamoTable
      Events:
        GetPet:
          Type: Api
          Properties:
            Path: /pets/{petId}
            Method: get
            RestApiId:
              Ref: ApiGatewayApi
  ListFunction:
    Type: AWS::Serverless::Function
    Properties:
      Runtime: dotnet6
      Handler: PetsApi::PetsApi.Function::List
      CodeUri: ./src/PetsApi/
      Policies:
        - DynamoDBCrudPolicy:
            TableName:
              Ref: DynamoTable
      Events:
        ListPet:
          Type: Api
          Properties:
            Path: /pets
            Method: get
            RestApiId:
              Ref: ApiGatewayApi

  EditFunction:
    Type: AWS::Serverless::Function
    Properties:
      Runtime: dotnet6
      Handler: PetsApi::PetsApi.Function::Edit
      CodeUri: ./src/PetsApi/
      Policies:
        - DynamoDBCrudPolicy:
            TableName:
              Ref: DynamoTable
      Events:
        EditPet:
          Type: Api
          Properties:
            Path: /pets/{petId}
            Method: put
            RestApiId:
              Ref: ApiGatewayApi
Outputs:
  PetsApi:
    Description: "API Gateway endpoint URL"
    Value: 
      Fn::Sub: "https://${ApiGatewayApi}.execute-api.${AWS::Region}.amazonaws.com/Prod/pets/"
```

The application consists of four Lambda functions for registering, editing, listing, and retrieving pets from storage (DynamoDB). All the Lambda functions are accessible through an API Gateway, where we have added CORS settings to allow access from any origin (we do not recommend this for real applications). Deploy the application running `sam build` and `sam deploy --guided`.

## OpenAPI Specification

In the solution, navigate to the `openapi.yaml` file:

```yaml
openapi: "3.0.1"
info:
  title: "petsapi"
  version: "1.0"
paths:
  /pets:
    get:
      summary: List pets
      responses:
        200:
          description: Successful operation
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ArrayOfPet' 
    post:
      summary: Register pet
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/RegisterPetRequest'
        required: true
      responses:
        200:
          description: Successful operation
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/RegisterPetResponse' 
  /pets/{petId}:
    get:
      summary: Get pet
      parameters:
      - name: "petId"
        in: "path"
        required: true
        schema:
          type: "string"
      responses:
        200:
          description: Successful operation
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Pet'
        400:
          description: Not found
    put:
      summary: Edit pet
      parameters:
      - name: "petId"
        in: "path"
        required: true
        schema:
          type: "string"
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/EditPetRequest'
        required: true
      responses:
        200:
          description: Successful operation
        400:
          description: Not found
components:
  schemas:
    ArrayOfPet:
      type: "array"
      items:
        $ref: "#/components/schemas/Pet"
    RegisterPetRequest:
      type: object
      properties:
        type:
          type: string
        name:
          type: string
      required:
        - type
        - name
    RegisterPetResponse:
      type: object
      properties:
        petId:
          type: string
      required:
        - petId
    EditPetRequest:
      type: object
      properties:
        type:
          type: string
        name:
          type: string
      required:
        - name
        - type
    Pet:
      type: object
      properties:
        petId:
          type: string
        type:
          type: string
        name:
          type: string
      required:
        - petId
        - type
        - name
```

Navigate to [https://editor.swagger.io/](https://editor.swagger.io/) and paste the content above to see a visual representation of the OpenAPI specification:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1701887295775/a9b72d7d-6455-48fe-9aa2-e240a81f47ee.png align="center")

## API Gateway Integration

API Gateway can be configured using an OpenAPI specification that includes a set of [extensions](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-swagger-extensions.html) to capture most of its specific properties, such as integration with Lambda functions. The easiest way to set a custom OpenAPI specification is by modifying the existing one that is generated by default. Run the following command to export the OpenAPI specification from the API Gateway:

```powershell
aws apigateway get-export --parameters extensions='apigateway' --rest-api-id <rest_api_id>--stage-name Prod --export-type oas30 openapi_with_extensions.yaml --accepts application/yaml
```

The file contents should look something like this:

```yaml
openapi: "3.0.1"
info:
  title: "petsapi"
  version: "1.0"
servers:
- url: "https://ecpgr6wvb6.execute-api.us-east-2.amazonaws.com/{basePath}"
  variables:
    basePath:
      default: "Prod"
paths:
  /pets:
    get:
      x-amazon-apigateway-integration:
        type: "aws_proxy"
        httpMethod: "POST"
        uri: "arn:aws:apigateway:us-east-2:lambda:path/2015-03-31/functions/arn:aws:lambda:us-east-2:025381531841:function:petsapi-ListFunction-QNclQ8uvBp8f/invocations"
        passthroughBehavior: "when_no_match"
    post:
      x-amazon-apigateway-integration:
        type: "aws_proxy"
        httpMethod: "POST"
        uri: "arn:aws:apigateway:us-east-2:lambda:path/2015-03-31/functions/arn:aws:lambda:us-east-2:025381531841:function:petsapi-RegisterFunction-h7T1XcNCNd30/invocations"
        passthroughBehavior: "when_no_match"
    options:
      responses:
        "200":
          description: "200 response"
          headers:
            Access-Control-Allow-Origin:
              schema:
                type: "string"
            Access-Control-Allow-Methods:
              schema:
                type: "string"
            Access-Control-Allow-Headers:
              schema:
                type: "string"
          content: {}
      x-amazon-apigateway-integration:
        type: "mock"
        responses:
          default:
            statusCode: "200"
            responseParameters:
              method.response.header.Access-Control-Allow-Methods: "'*'"
              method.response.header.Access-Control-Allow-Headers: "'*'"
              method.response.header.Access-Control-Allow-Origin: "'*'"
            responseTemplates:
              application/json: "{}\n"
        requestTemplates:
          application/json: "{\n  \"statusCode\" : 200\n}\n"
        passthroughBehavior: "when_no_match"
  /pets/{petId}:
    get:
      parameters:
      - name: "petId"
        in: "path"
        required: true
        schema:
          type: "string"
      x-amazon-apigateway-integration:
        type: "aws_proxy"
        httpMethod: "POST"
        uri: "arn:aws:apigateway:us-east-2:lambda:path/2015-03-31/functions/arn:aws:lambda:us-east-2:025381531841:function:petsapi-GetFunction-wpx1KVbQl6fj/invocations"
        passthroughBehavior: "when_no_match"
    put:
      parameters:
      - name: "petId"
        in: "path"
        required: true
        schema:
          type: "string"
      x-amazon-apigateway-integration:
        type: "aws_proxy"
        httpMethod: "POST"
        uri: "arn:aws:apigateway:us-east-2:lambda:path/2015-03-31/functions/arn:aws:lambda:us-east-2:025381531841:function:petsapi-EditFunction-2iNi97wo5HYK/invocations"
        passthroughBehavior: "when_no_match"
    options:
      parameters:
      - name: "petId"
        in: "path"
        required: true
        schema:
          type: "string"
      responses:
        "200":
          description: "200 response"
          headers:
            Access-Control-Allow-Origin:
              schema:
                type: "string"
            Access-Control-Allow-Methods:
              schema:
                type: "string"
            Access-Control-Allow-Headers:
              schema:
                type: "string"
          content: {}
      x-amazon-apigateway-integration:
        type: "mock"
        responses:
          default:
            statusCode: "200"
            responseParameters:
              method.response.header.Access-Control-Allow-Methods: "'*'"
              method.response.header.Access-Control-Allow-Headers: "'*'"
              method.response.header.Access-Control-Allow-Origin: "'*'"
            responseTemplates:
              application/json: "{}\n"
        requestTemplates:
          application/json: "{\n  \"statusCode\" : 200\n}\n"
        passthroughBehavior: "when_no_match"
components: {}
```

The presence of the [`x-amazon-apigateway-integration`](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-swagger-extensions-integration.html) extension specifies the integration used for each method against the backend.

* `type`: The type of integration with the backend. Use `aws_proxy` for integration with AWS Lambda functions.
    
* `httpMethod`: The HTTP method used in the integration request. For Lambda function invocations, the value must be `POST`.
    
* `uri`: The endpoint URI of the backend. For integrations of the `aws_proxy` type, the value should be `arn:aws:apigateway:<region_arn>:lambda:path/2015-03-31/functions/<lambda_arn>/invocations`.
    
* `passthroughBehavior`: Specifies how a payload (with an unmapped content type) is passed through the integration request without any changes.
    

So, let's begin by replacing all the static `uri`s with their corresponding values based on the Lambda functions defined in the `template.yaml` file:

```yaml
openapi: "3.0.1"
info:
  title: "petsapi"
  version: "1.0"
paths:
  /pets:
    get:
      x-amazon-apigateway-integration:
        type: "aws_proxy"
        httpMethod: "POST"
        uri:
          Fn::Sub: "arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${ListFunction.Arn}/invocations"
        passthroughBehavior: "when_no_match"
    post:
      x-amazon-apigateway-integration:
        type: "aws_proxy"
        httpMethod: "POST"
        httpMethod: "POST"
        uri:
          Fn::Sub: "arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${RegisterFunction.Arn}/invocations"
        passthroughBehavior: "when_no_match"
    options:
      responses:
        "200":
          description: "200 response"
          headers:
            Access-Control-Allow-Origin:
              schema:
                type: "string"
            Access-Control-Allow-Methods:
              schema:
                type: "string"
            Access-Control-Allow-Headers:
              schema:
                type: "string"
          content: {}
      x-amazon-apigateway-integration:
        type: "mock"
        responses:
          default:
            statusCode: "200"
            responseParameters:
              method.response.header.Access-Control-Allow-Methods: "'*'"
              method.response.header.Access-Control-Allow-Headers: "'*'"
              method.response.header.Access-Control-Allow-Origin: "'*'"
            responseTemplates:
              application/json: "{}\n"
        requestTemplates:
          application/json: "{\n  \"statusCode\" : 200\n}\n"
        passthroughBehavior: "when_no_match"
  /pets/{petId}:
    get:
      parameters:
      - name: "petId"
        in: "path"
        required: true
        schema:
          type: "string"
      x-amazon-apigateway-integration:
        type: "aws_proxy"
        httpMethod: "POST"
        uri: 
          Fn::Sub: "arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${GetFunction.Arn}/invocations"
        passthroughBehavior: "when_no_match"
    put:
      parameters:
      - name: "petId"
        in: "path"
        required: true
        schema:
          type: "string"
      x-amazon-apigateway-integration:
        type: "aws_proxy"
        httpMethod: "POST"
        uri: 
          Fn::Sub: "arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${EditFunction.Arn}/invocations"
        passthroughBehavior: "when_no_match"
    options:
      parameters:
      - name: "petId"
        in: "path"
        required: true
        schema:
          type: "string"
      responses:
        "200":
          description: "200 response"
          headers:
            Access-Control-Allow-Origin:
              schema:
                type: "string"
            Access-Control-Allow-Methods:
              schema:
                type: "string"
            Access-Control-Allow-Headers:
              schema:
                type: "string"
          content: {}
      x-amazon-apigateway-integration:
        type: "mock"
        responses:
          default:
            statusCode: "200"
            responseParameters:
              method.response.header.Access-Control-Allow-Methods: "'*'"
              method.response.header.Access-Control-Allow-Headers: "'*'"
              method.response.header.Access-Control-Allow-Origin: "'*'"
            responseTemplates:
              application/json: "{}\n"
        requestTemplates:
          application/json: "{\n  \"statusCode\" : 200\n}\n"
        passthroughBehavior: "when_no_match"
components: {}
```

The next step involves merging the content of the `openapi.yaml` file with the text provided above:

```yaml
openapi: "3.0.1"
info:
  title: "petsapi"
  version: "1.0"
paths:
  /pets:
    get:
      summary: List pets
      x-amazon-apigateway-integration:
        httpMethod: "POST"
        uri:
          Fn::Sub: "arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${ListFunction.Arn}/invocations"
        passthroughBehavior: "when_no_match"
        type: "aws_proxy"
      responses:
        200:
          description: Successful operation
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ArrayOfPet' 
    post:
      summary: Register pet
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/RegisterPetRequest'
        required: true
      x-amazon-apigateway-integration:
        httpMethod: "POST"
        uri:
          Fn::Sub: "arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${RegisterFunction.Arn}/invocations"
        passthroughBehavior: "when_no_match"
        type: "aws_proxy"
      responses:
        200:
          description: Successful operation
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/RegisterPetResponse' 
  /pets/{petId}:
    get:
      summary: Get pet
      parameters:
      - name: "petId"
        in: "path"
        required: true
        schema:
          type: "string"
      x-amazon-apigateway-integration:
        httpMethod: "POST"
        uri: 
          Fn::Sub: "arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${GetFunction.Arn}/invocations"
        passthroughBehavior: "when_no_match"
        type: "aws_proxy"
      responses:
        200:
          description: Successful operation
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Pet'
        400:
          description: Not found
    put:
      summary: Edit pet
      parameters:
      - name: "petId"
        in: "path"
        required: true
        schema:
          type: "string"
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/EditPetRequest'
        required: true
      x-amazon-apigateway-integration:
        httpMethod: "POST"
        uri: 
          Fn::Sub: "arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${EditFunction.Arn}/invocations"
        passthroughBehavior: "when_no_match"
        type: "aws_proxy"
      responses:
        200:
          description: Successful operation
        400:
          description: Not found
components:
  schemas:
    ArrayOfPet:
      type: "array"
      items:
        $ref: "#/components/schemas/Pet"
    RegisterPetRequest:
      type: object
      properties:
        type:
          type: string
        name:
          type: string
      required:
        - type
        - name
    RegisterPetResponse:
      type: object
      properties:
        petId:
          type: string
      required:
        - petId
    EditPetRequest:
      type: object
      properties:
        type:
          type: string
        name:
          type: string
      required:
        - name
        - type
    Pet:
      type: object
      properties:
        petId:
          type: string
        type:
          type: string
        name:
          type: string
      required:
        - petId
        - type
        - name
```

The final step is to modify the `ApiGatewayApi` resource in the `template.yaml` file:

```yaml
  ApiGatewayApi:
    Type: AWS::Serverless::Api
    Properties:
      Name: petsapi
      StageName: Prod
      Cors:
        AllowMethods: "'*'"
        AllowHeaders: "'*'"
        AllowOrigin: "'*'"
      OpenApiVersion: '3.0.1'
      DefinitionBody:
        'Fn::Transform':
          Name: 'AWS::Include'
          Parameters:
            Location: openapi_with_extensions.yaml
```

The `Fn::Transform` and `AWS::Include` directives instruct to retrieve the file from the file system and dynamically evaluate all items before inserting them into the final template that is executed.

## Testing the Integration

Now that we have our specification ready run `sam build` and `sam deploy --guided` to deploy the application and `aws apigateway get-export --parameters extensions='apigateway' --rest-api-id <rest_api_id>--stage-name Prod --export-type oas30 openapi_specification.yaml --accepts application/yaml` to download the new version of the OpenAPI specification generated by API gateway. Copy the content of the file into [https://editor.swagger.io/](https://editor.swagger.io/):

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1701902324670/22ca9ebd-5f6b-4176-a5e8-a0391f5e4532.png align="center")

Due to the CORS configuration we are using, we can test the API directly from the SwaggerUI itself. In conclusion, by following the steps presented in this article, we can effectively document your AWS Lambda functions using the OpenAPI specification, making it easier for humans and machines to understand and interact with your service. The final code can be found [here](https://github.com/raulnq/aws-lambda-openapi/tree/openapi). Thanks, and happy coding.