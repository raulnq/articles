---
title: "Building extensions for AWS Lambda with .NET"
datePublished: Mon Apr 03 2023 03:46:53 GMT+0000 (Coordinated Universal Time)
cuid: clg0aiau6000709iebpjs054v
slug: building-extensions-for-aws-lambda-with-net
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1680301976516/e510dbae-6c9d-4567-a1fb-027aedcf2c0f.png
tags: aws, net, aws-lambda

---

AWS Lambda extensions solve the problem of having to include additional code or dependencies within the Lambda function itself. Instead, Lambda extensions are separate processes that can be loaded into the same runtime environment as the Lambda function, allowing for additional functionality and customization without affecting the function's code or performance. This can simplify the development process and improve the overall manageability of Lambda functions. Some examples of use cases for Lambda extensions include:

* Customizing the runtime environment, like fetching configuration settings or secrets before the function's invocation.
    
* Improving observability, capturing diagnostic information before, during and after function invocation.
    
* Integrating functions with other monitoring, observability, security, and governance tools.
    

We can find information about AWS Lambda extensions [here](https://docs.aws.amazon.com/lambda/latest/dg/runtimes-extensions-api.html). Now, let's see how to build an extension using .NET.

## **Prerequisites**

* An [**IAM User**](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html#id_users_create_console) with programmatic access.
    
* Install [**AWS CLI**](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-creds).
    
* Install the Amazon Lambda Templates (`dotnet new -i Amazon.Lambda.Templates`).
    
* Install the Amazon Lambda Tools (`dotnet tool install -g` [`Amazon.Lambda.Tools`](http://Amazon.Lambda.Tools)).
    

## **Creating a Lambda Function**

Run the following commands to create a Lambda function:

```bash
dotnet new lambda.EmptyFunction -n HelloWorldApi -o .
dotnet add src/HelloWorldApi package Amazon.Lambda.APIGatewayEvents
dotnet new sln -n DotnetExtensionSandbox
dotnet sln add --in-root src/HelloWorldApi
```

Open the solution, go to the `HelloWorldApi` project, and update the `Function.cs` file as follows:

```csharp
using Amazon.Lambda.APIGatewayEvents;
using Amazon.Lambda.Core;

[assembly: LambdaSerializer(typeof(Amazon.Lambda.Serialization.SystemTextJson.DefaultLambdaJsonSerializer))]
namespace HelloWorldApi
{
    public class Function
    {
        public async Task<APIGatewayHttpApiV2ProxyResponse> FunctionHandler(APIGatewayHttpApiV2ProxyRequest input, ILambdaContext context)
        {
            return new APIGatewayHttpApiV2ProxyResponse
            {
                Body = @"{""Message"":""Hello world!""}",
                StatusCode = 200,
                Headers = new Dictionary<string, string> { { "Content-Type", "application/json" } }
            };
        }
    }
}
```

## **How does it work?**

A Lambda extension can be deployed as a [Lambda layer](https://blog.raulnq.com/aws-lambda-layers-in-net). When the Lambda service starts an execution environment, it extracts the content of the layer into the `/opt` directory. Lambda then looks for any extensions in the `/opt/extensions` directory and starts initializing them. Extensions need to be executable as binaries or scripts and can be explained with the following pseudo-code:

```csharp
extentions_api.register();
while (true) {
    response = extentions_api.next();
    if (response.type == “shutdown”)
    {
        shutdown_code();
        break;
    }
    do_something();
}
```

During the `init` phase, each extension needs to register with Lambda to receive events (`invoke` and `shutdown`). Lambda uses the full file name of the extension to validate that the extension has completed the bootstrap sequence. Therefore, each `Register` API call must include the `Lambda-Extension-Name` header with the full file name of the extension. We will receive the `Lambda-Extension-Identifier` header that is required for all subsequent requests.

To process events during the `invoke` phase, we need to define a loop and call the Next API (this is a blocking call) until a `shutdown` event is received. When a Lambda function is invoked sends an event to each extension that is registered for the event.

When Lambda is about to shut down sends a `shutdown` event to each registered extension. Extensions can use this time for final cleanup tasks.

## **Creating a Lambda Extension**

Run the following commands to create a Lambda extension:

```bash
dotnet new console -n MyExtension -o src/MyExtension
dotnet sln add --in-root src/MyExtension
```

Go to the `MyExtension` project and create a `Payload.cs` file with the following content:

```csharp
namespace MyExtension
{
    public enum EventType
    {
        INVOKE,
        SHUTDOWN,
    }

    public class Payload
    {
        public EventType EventType { get; set; }
        public string? RequestId { get; set; }
        public string? InvokedFunctionArn { get; set; }
        public decimal DeadlineMs { get; set; }
        public string? ShutdownReason { get; set; }
    }
}
```

Create an `Extension.cs` file as follows:

```csharp
using System.Text;
using System.Text.Json.Serialization;
using System.Text.Json;

namespace MyExtension
{
    public class Extension
    {
        private readonly string _name;

        private readonly Uri _registerUrl;

        private readonly Uri _nextUrl;

        private readonly Uri _subscriptionUrl;

        public Extension(string? name)
        {
            _name = name ?? throw new ArgumentNullException(nameof(name), "Extension name cannot be null");

            var apiUri = new UriBuilder(Environment.GetEnvironmentVariable("AWS_LAMBDA_RUNTIME_API")!).Uri;

            _registerUrl = new Uri(apiUri, $"2020-01-01/extension/register");

            _nextUrl = new Uri(apiUri, $"2020-01-01/extension/event/next");

            _subscriptionUrl = new Uri(apiUri, $"2022-07-01/telemetry");
        }

        private async Task<string> Register(HttpClient httpClient)
        {
            var options = new JsonSerializerOptions();

            options.Converters.Add(new JsonStringEnumConverter());

            using var content = new StringContent(JsonSerializer.Serialize(new { events = new EventType[] { EventType.INVOKE, EventType.SHUTDOWN } }, options), Encoding.UTF8, "application/json");

            content.Headers.Add("Lambda-Extension-Name", _name);

            using (var response = await httpClient.PostAsync(_registerUrl, content))
            {
                if (!response.IsSuccessStatusCode)
                {
                    Console.WriteLine($"[{_name}] Error response received for registration request: {await response.Content.ReadAsStringAsync()}");
                    response.EnsureSuccessStatusCode();
                }

                var id = response.Headers.GetValues("Lambda-Extension-Identifier").FirstOrDefault();

                if (string.IsNullOrEmpty(id))
                {
                    throw new ApplicationException("Extension API register call didn't return a valid identifier.");
                }

                return id;
            }
        }

        public async Task Start()
        {
            using (var httpClient = new HttpClient() { Timeout = Timeout.InfiniteTimeSpan })
            {
                var id = await Register(httpClient);

                Console.WriteLine($"[{_name}] Registered extension with id = {id}");

                httpClient.DefaultRequestHeaders.Add("Lambda-Extension-Identifier", id);

                while (true)
                {
                    var payload = await GetNext(httpClient);

                    if(payload.EventType== EventType.SHUTDOWN)
                    {
                        Console.WriteLine($"[{_name}] Shutting down extension: {payload.ShutdownReason}");
                        break;
                    }

                    Console.WriteLine($"[{_name}] Handling invoke from extension: {payload.RequestId}");
                }
            }
        }

        private async Task<Payload> GetNext(HttpClient httpClient)
        {
            var contentBody = await httpClient.GetStringAsync(_nextUrl);

            var options = new JsonSerializerOptions() { PropertyNameCaseInsensitive = true };

            options.Converters.Add(new JsonStringEnumConverter());

            return JsonSerializer.Deserialize<Payload>(contentBody, options)!;
        }
    }
}
```

Update the `Program.cs` file as follows:

```csharp
using System.Reflection;

namespace MyExtension
{
    class Program
    {
        static async Task Main(string[] args)
        {
            var name = (1 == args.Length)
                ? args[0]
                : Assembly.GetEntryAssembly()?.GetName()?.Name;

            await new Extension(name).Start();
        }
    }
}
```

## Building and Deploying

Create a `template.xml` file ([**AWS SAM**](https://blog.raulnq.com/deploying-aws-lambda-functions-with-aws-sam)) with the following content:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: >
  Lambda Extensions

Resources:   

  MyExtensionLayer:
    Type: AWS::Serverless::LayerVersion
    Properties:
      ContentUri: ./publish/

  MyFunction:
    Type: AWS::Serverless::Function
    Properties:
      Timeout: 60
      MemorySize: 512
      Runtime: dotnet6
      Layers:
        - !Ref MyExtensionLayer
      Architectures:
        - x86_64    
      Handler: HelloWorldApi::HelloWorldApi.Function::FunctionHandler
      CodeUri: ./src/HelloWorldApi/
      Events:
        ListPosts:
          Type: Api
          Properties:
            Path: /
            Method: get
```

Run the following command for building and packaging the extension:

```bash
dotnet publish ./src/MyExtension/MyExtension.csproj --configuration Release --runtime linux-x64 -p:PublishSingleFile=true -p:PublishTrimmed=true -p:DebugSymbols=false -p:DebugType=None --self-contained true --output ./publish/extensions
```

Run `sam build` and `sam deploy --guided` and follow the instructions to deploy the Lambda function and extension. Check the Lambda function logs to see the outputs written by our extension.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1680489741276/51cbe190-0f29-405e-9bfe-144ef47b981d.png align="center")

The code is available [here](https://github.com/raulnq/aws-lambda-extensions). Thanks, and happy coding.