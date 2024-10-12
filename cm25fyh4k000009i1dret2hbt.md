---
title: "How to Use WebSockets with Amazon API Gateway"
datePublished: Sat Oct 12 2024 00:51:48 GMT+0000 (Coordinated Universal Time)
cuid: cm25fyh4k000009i1dret2hbt
slug: how-to-use-websockets-with-amazon-api-gateway
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1728492707525/8abb240e-01aa-49c9-9c07-d1336a458777.png
tags: websockets, net, aws-lambda, aws-apigateway

---

[WebSockets](https://en.wikipedia.org/wiki/WebSocket) is a communication protocol that enables full-duplex data exchange over a single TCP connection, facilitating real-time interactions between clients and servers. Amazon API Gateway supports this protocol through the WebSockets API, relieving us of the responsibility to manage the connection between the client and the server.

A WebSocket API consists of one or more [**routes**](https://docs.aws.amazon.com/apigateway/latest/developerguide/websocket-api-develop-routes.html), each defining how a message is processed and [**integrated**](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-websocket-api-integrations.html) with a backend service, typically a Lambda function. Each route has a **key** that must match the result of the **selection expression** when a message is received to determine its usage. This **selection expression** specifies a JSON property expected in the message, and usually looks like `$request.body.{path_to_body_element}`. By default, three routes are provided:

* [`$connect`](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-websocket-api-route-keys-connect-disconnect.html): Triggered when a connection between the client and the WebSocket API is initiated. Once connected, a unique connection ID is assigned to the client and included in every request to the backend service. This connection ID can be used to send messages back to clients using an endpoint provided by Amazon API Gateway.
    
* [`$disconnect`](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-websocket-api-route-keys-connect-disconnect.html): Triggered when the client disconnects from the WebSocket API.
    
* [`$default`](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-websocket-api-routes-integrations.html): Used when no matching route is found.
    

In addition to these routes, custom routes can be defined using a key and integration of your choice. To better understand these concepts, let's build a basic chat application, similar to what we have already done in:

* [Getting Started with SignalR in .NET 6](https://blog.raulnq.com/getting-started-with-signalr-in-net-6)
    
* [Using Raw WebSockets in .NET](https://blog.raulnq.com/using-raw-websockets-in-net)
    

## Pre-requisites

* Have an [IAM User](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html#id_users_create_console) with programmatic access.
    
* Install the Amazon Lambda Templates (`dotnet new -i Amazon.Lambda.Templates`)
    
* Install the Amazon Lambda Tools (`dotnet tool install -g Amazon.Lambda.Tools`)
    
* Install [AWS SAM CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html).
    

## The Backend Services

Run the following commands to set up our Lambda functions:

```powershell
dotnet new lambda.EmptyFunction -n MyLambda -o .
dotnet add src/MyLambda package Amazon.ApiGatewayManagementApi
dotnet add src/MyLambda package AWSSDK.DynamoDBv2
dotnet add src/MyLambda package Amazon.Lambda.APIGatewayEvents
dotnet new sln -n MyChatApp
dotnet sln add --in-root src/MyLambda
```

Open the `Program.cs` file and update the content as follows:

```csharp
using Amazon.ApiGatewayManagementApi;
using Amazon.ApiGatewayManagementApi.Model;
using Amazon.DynamoDBv2;
using Amazon.DynamoDBv2.Model;
using Amazon.Lambda.APIGatewayEvents;
using Amazon.Lambda.Core;
using System.Text;
using System.Text.Json;

[assembly: LambdaSerializer(typeof(Amazon.Lambda.Serialization.SystemTextJson.DefaultLambdaJsonSerializer))]
namespace MyLambda;

public class Function
{
    private AmazonDynamoDBClient _amazonDynamoDB;
    private string _table;
    public record Payload(string Message);
    private readonly JsonSerializerOptions _options;

    public Function()
    {
        _amazonDynamoDB = new AmazonDynamoDBClient();
        _table = "connections";
        _options = new JsonSerializerOptions()
        {
            PropertyNamingPolicy = JsonNamingPolicy.CamelCase,
        };
    }

    public async Task<APIGatewayProxyResponse> Connect(APIGatewayProxyRequest input, ILambdaContext context)
    {
        var putItemRequest = new PutItemRequest
        {
            TableName = _table,
            Item = new Dictionary<string, AttributeValue> 
            {
                { "connectionid", new AttributeValue { S = input.RequestContext.ConnectionId } }
            }
        };

        await _amazonDynamoDB.PutItemAsync(putItemRequest);

        return new APIGatewayProxyResponse
        {
            Body = "connected",
            StatusCode = 200,
            Headers = new Dictionary<string, string> { { "Content-Type", "application/json" } }
        };
    }

    public async Task<APIGatewayProxyResponse> Disconnect(APIGatewayProxyRequest input, ILambdaContext context)
    {
        var deleteItemRequest = new DeleteItemRequest
        {
            TableName = _table,
            Key = new Dictionary<string, AttributeValue> 
            {
                { "connectionid", new AttributeValue { S = input.RequestContext.ConnectionId } }
            }
        };

        await _amazonDynamoDB.DeleteItemAsync(deleteItemRequest);

        return new APIGatewayProxyResponse
        {
            Body = "disconnected",
            StatusCode = 200,
            Headers = new Dictionary<string, string> { { "Content-Type", "application/json" } }
        };
    }

    public async Task<APIGatewayProxyResponse> Send(APIGatewayProxyRequest input, ILambdaContext context)
    {
        var scanRequest = new ScanRequest
        {
            TableName = _table,
        };

        var scanResponse = await _amazonDynamoDB.ScanAsync(scanRequest);
        var apiClient = new AmazonApiGatewayManagementApiClient(new AmazonApiGatewayManagementApiConfig
        {
            ServiceURL = $"https://{input.RequestContext.DomainName}/{input.RequestContext.Stage}"
        });

        var payload = JsonSerializer.Deserialize<Payload>(input.Body, _options)!;
        var message = Encoding.UTF8.GetBytes($"{input.RequestContext.ConnectionId} says {payload.Message}");
        foreach (var item in scanResponse.Items)
        {
            var connectionId = item["connectionid"].S;
            var postMessageRequest = new PostToConnectionRequest
            {
                ConnectionId = connectionId,
                Data = new MemoryStream(message)
            };

            try
            {
                await apiClient.PostToConnectionAsync(postMessageRequest);
            }
            catch (GoneException)
            {
                var deleteItemRequest = new DeleteItemRequest
                {
                    TableName = _table,
                    Key = new Dictionary<string, AttributeValue> 
                    {
                        { "connectionid", new AttributeValue { S = input.RequestContext.ConnectionId }}
                    }
                };

                await _amazonDynamoDB.DeleteItemAsync(deleteItemRequest);
            }
        }

        return new APIGatewayProxyResponse
        {
            Body = "message sent",
            StatusCode = 200,
            Headers = new Dictionary<string, string> { { "Content-Type", "application/json" } }
        };
    }
}
```

We are defining the following methods:

* `Connect`: This method saves the connection ID in the `connections` DynamoDB table.
    
* `Disconnect`: This method deletes the connection ID from the `connections` DynamoDB table.
    
* `Send:` This method lists all the connection IDs and broadcasts the incoming message to each one.
    

## AWS SAM template

At the solution level, create a `template.yml` file with the following content:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: >
  SAM

Resources:
  ConnectionsTable:
    Type: AWS::Serverless::SimpleTable
    Properties:
      PrimaryKey:
        Name: connectionid
        Type: String
      TableName: connections

  ChatApi:
    Type: AWS::ApiGatewayV2::Api
    Properties:
      Name: MyChatApp
      ProtocolType: WEBSOCKET
      RouteSelectionExpression: "$request.body.action"

  ConnectFunction:
    Type: AWS::Serverless::Function
    Properties:
      Timeout: 60
      MemorySize: 512
      Tracing: Active
      Runtime: dotnet8
      Architectures:
        - x86_64    
      Handler: MyLambda::MyLambda.Function::Connect
      CodeUri: ./src/MyLambda/
      Policies:
        - Statement:
          - Effect: Allow
            Action:
              - 'dynamodb:PutItem'
            Resource:
              - !Sub 'arn:${AWS::Partition}:dynamodb:${AWS::Region}:${AWS::AccountId}:table/${ConnectionsTable}'

  ConnectRoute:
    Type: AWS::ApiGatewayV2::Route
    Properties:
      ApiId: !Ref ChatApi
      RouteKey: $connect
      AuthorizationType: NONE
      OperationName: Connect Route
      Target: !Sub "integrations/${ConnectIntegration}"

  ConnectIntegration:
    Type: AWS::ApiGatewayV2::Integration
    Properties:
      ApiId: !Ref ChatApi
      Description: Connect Integration
      IntegrationType: AWS_PROXY
      IntegrationUri: !Sub "arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${ConnectFunction.Arn}/invocations"

  ConnectFunctionPermission:
    Type: AWS::Lambda::Permission
    DependsOn:
      - ChatApi
    Properties:
      Action: lambda:InvokeFunction
      FunctionName: !Ref ConnectFunction
      Principal: apigateway.amazonaws.com

  DisconnectFunction:
    Type: AWS::Serverless::Function
    Properties:
      Timeout: 60
      MemorySize: 512
      Tracing: Active
      Runtime: dotnet8
      Architectures:
        - x86_64    
      Handler: MyLambda::MyLambda.Function::Disconnect
      CodeUri: ./src/MyLambda/
      Policies:
        - Statement:
          - Effect: Allow
            Action:
              - 'dynamodb:DeleteItem'
            Resource:
              - !Sub 'arn:${AWS::Partition}:dynamodb:${AWS::Region}:${AWS::AccountId}:table/${ConnectionsTable}'

  DisconnectRoute:
    Type: AWS::ApiGatewayV2::Route
    Properties:
      ApiId: !Ref ChatApi
      RouteKey: $disconnect
      AuthorizationType: NONE
      OperationName: Disconnect Route
      Target: !Sub "integrations/${DisconnectIntegration}"

  DisconnectIntegration:
    Type: AWS::ApiGatewayV2::Integration
    Properties:
      ApiId: !Ref ChatApi
      Description: Disconnect Integration
      IntegrationType: AWS_PROXY
      IntegrationUri: !Sub "arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${DisconnectFunction.Arn}/invocations"

  DisconnectFunctionPermission:
    Type: AWS::Lambda::Permission
    DependsOn:
      - ChatApi
    Properties:
      Action: lambda:InvokeFunction
      FunctionName: !Ref DisconnectFunction
      Principal: apigateway.amazonaws.com

  SendFunction:
    Type: AWS::Serverless::Function
    Properties:
      Timeout: 60
      MemorySize: 512
      Tracing: Active
      Runtime: dotnet8
      Architectures:
        - x86_64    
      Handler: MyLambda::MyLambda.Function::Send
      CodeUri: ./src/MyLambda/
      Policies:
        - Statement:
          - Effect: Allow
            Action:
              - 'dynamodb:Scan'
            Resource:
              - !Sub 'arn:${AWS::Partition}:dynamodb:${AWS::Region}:${AWS::AccountId}:table/${ConnectionsTable}'
          - Effect: Allow
            Action:
              - 'execute-api:ManageConnections'
            Resource:
              - !Sub 'arn:aws:execute-api:${AWS::Region}:${AWS::AccountId}:${ChatApi}/*'

  SendRoute:
    Type: AWS::ApiGatewayV2::Route
    Properties:
      ApiId: !Ref ChatApi
      RouteKey: send
      AuthorizationType: NONE
      OperationName: Send Route
      Target: !Sub "integrations/${SendIntegration}"

  SendIntegration:
    Type: AWS::ApiGatewayV2::Integration
    Properties:
      ApiId: !Ref ChatApi
      Description: Send Integration
      IntegrationType: AWS_PROXY
      IntegrationUri: !Sub "arn:aws:apigateway:${AWS::Region}:lambda:path/2015-03-31/functions/${SendFunction.Arn}/invocations"

  SendFunctionPermission:
    Type: AWS::Lambda::Permission
    DependsOn:
      - ChatApi
    Properties:
      Action: lambda:InvokeFunction
      FunctionName: !Ref SendFunction
      Principal: apigateway.amazonaws.com

  Deployment:
    Type: AWS::ApiGatewayV2::Deployment
    DependsOn:
    - ConnectRoute
    - DisconnectRoute
    - SendRoute
    Properties:
      ApiId: !Ref ChatApi

  Stage:
    Type: AWS::ApiGatewayV2::Stage
    Properties:
      StageName: prod
      Description: prod stage
      DeploymentId: !Ref Deployment
      ApiId: !Ref ChatApi
      AutoDeploy: true

Outputs:

  WebSocketURI:
    Description: "The WSS protocol URI to connect to"
    Value: !Sub wss://${ChatApi}.execute-api.${AWS::Region}.amazonaws.com/${Stage}
```

In the script above, we define a DynamoDB table using the [`AWS::Serverless::SimpleTable`](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-resource-simpletable.html) resource and the WebSocket API through the [`AWS::ApiGatewayV2::Api`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-apigatewayv2-api.html) resource. For each route - `$connect`, `$disconnect`, and `send` - we create the following resources:

* [`AWS::Serverless::Function`](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-resource-function.html): Creates the Lambda function with the necessary permissions.
    
* [`AWS::Lambda::Permission`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-lambda-permission.html): Grants permission for Amazon API Gateway to invoke the Lambda function.
    
* [`AWS::ApiGatewayV2::Route`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-apigatewayv2-route.html#cfn-apigatewayv2-route-operationname): Creates a route with a specific `RouteKey`.
    
* [`AWS::ApiGatewayV2::Integration`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-apigatewayv2-integration.html): Creates the connection between the route and the Lambda function.
    

Finally, the [`AWS::ApiGatewayV2::Deployment`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-apigatewayv2-deployment.html) and [`AWS::ApiGatewayV2::Stage`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-apigatewayv2-stage.html) resources complete the setup for our application environment. Run the following commands to deploy the resources to AWS:

```powershell
sam build
sam deploy --guided
```

## The Client

Create a `client.html` file at the solution level with the following content:

```xml
<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8">
    <title>Chat</title>
</head>
<body>
    <input id="message" placeholder="Message" />
    <button id="send" type="button">Send</button>
    <hr />
    <ul id="messages"></ul>
    <script>
        document.addEventListener("DOMContentLoaded", () => {
            const websocket = new WebSocket("[WebSocketURI]");
            websocket.onmessage = (e) => {
                const li = document.createElement("li");
                li.textContent = `${e.data}`;
                document.getElementById("messages").appendChild(li);
            };
            document.getElementById("send").addEventListener("click", async () => {
                const message = document.getElementById("message").value;
                const payload = { action: "send", message: message };
                try {
                    await websocket.send(JSON.stringify(payload));
                } catch (err) {
                    console.error(err);
                }
            });
        });
    </script>
</body>
</html>
```

The script above starts the WebSocket and sets up a handler to receive messages from the server. It also sets up a handler for the button to send messages to the server. You can find all the code [here](https://github.com/raulnq/aws-api-gateway-websockets). Thanks, and happy coding.