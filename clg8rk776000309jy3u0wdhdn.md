---
title: "Real-time log processing with AWS Lambda Telemetry API and .NET"
datePublished: Sun Apr 09 2023 02:06:24 GMT+0000 (Coordinated Universal Time)
cuid: clg8rk776000309jy3u0wdhdn
slug: real-time-log-processing-with-aws-lambda-telemetry-api-and-net
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1680918008347/d40814a6-3b3e-4d05-bc9a-9f2983e51f47.png
tags: aws, net, aws-lambda

---

While CloudWatch is an excellent tool for collecting and analyzing logs, countless third-party tools offer features we may want to use. So we needed a way to move CloudWatch logs into our favorite tool. For a long time, this was achieved by configuring a CloudWatch logs subscription filter to stream the logs to an additional Lambda function. The function can then process the logs and send them to the third-party tool using the appropriate API or SDK. But today, [AWS Lambda Telemetry API](https://docs.aws.amazon.com/lambda/latest/dg/runtimes-extensions-api.html) provides an easier way to receive telemetry data directly from the Lambda service and send it to custom destinations:

> Using the Lambda Telemetry API, your extensions can directly receive telemetry data from Lambda. During function initialization and invocation, Lambda automatically captures telemetry, such as logs, platform metrics, and platform traces. With Telemetry API, extensions can get this telemetry data directly from Lambda in near real-time.

To use the AWS telemetry API, we need to build a Lambda extension with extra features:

* Use the Telemetry API to subscribe to desired telemetry event. There are three types Platform telemetry, Function logs and Extension logs.
    
* Starts a telemetry listener (local HTTP or TCP endpoint).
    

After that, the Lambda service sends telemetry data to your telemetry listener. We batch the telemetry data as it arrives, and during the `invoke` phase, we send it to the target location.

> During the invocation, external extensions run in parallel with the function. They also continue running after the function has been completed. This enables you to capture diagnostic information or to send logs, metrics, and traces to a location of your choice.

The following pseudo-code summarized what we said:

```csharp
uri= "http://sandbox.localdomain:8080";
start_http_server(uri, post);
extentions_api.register();
logs_api.subscribe();
while (true) {
    response = extentions_api.next();
    if (response.type == “shutdown”)
    {
        send_buffer();
        break;
    }
    send_buffer();
}

post(event) {
    buffer(event);
}
```

We will use our prior [post](https://blog.raulnq.com/building-extensions-for-aws-lambda-with-net) about how to build extensions using .NET as a starting point. Please, download the code located [here](https://github.com/raulnq/aws-lambda-extensions). The same prerequisites apply to this post, open solution, and change the `MyExtension.csproj` as follows:

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net6.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
  </PropertyGroup>
</Project>
```

Add a `Log.cs` file with the following content:

```csharp
namespace MyExtension
{
    public class Log
    {
        public string Time { get; set; } = null!;
        public string Type { get; set; } = null!;
        public string Record { get; set; } = null!;
    }
}
```

Update the `Extension.cs` file as follows:

```csharp
using System.Text;
using System.Text.Json.Serialization;
using System.Text.Json;
using System.Collections.Concurrent;
using Microsoft.Extensions.Options;

namespace MyExtension
{
    public class Extension
    {
        private readonly string _name;

        private readonly Uri _registerUrl;

        private readonly Uri _nextUrl;

        private readonly Uri _subscriptionUrl;

        private readonly Uri _targetUrl;

        public Extension(string? name)
        {
            _name = name ?? throw new ArgumentNullException(nameof(name), "Extension name cannot be null");

            var apiUri = new UriBuilder(Environment.GetEnvironmentVariable("AWS_LAMBDA_RUNTIME_API")!).Uri;

            _registerUrl = new Uri(apiUri, $"2020-01-01/extension/register");

            _nextUrl = new Uri(apiUri, $"2020-01-01/extension/event/next");

            _subscriptionUrl = new Uri(apiUri, $"2022-07-01/telemetry");

            _targetUrl = new Uri(Environment.GetEnvironmentVariable("TOOL_API_URL")!);
        }

        private async Task Subscribe(HttpClient httpClient)
        {
            var options = new JsonSerializerOptions();

            options.Converters.Add(new JsonStringEnumConverter());

            var body = @"
{
   ""schemaVersion"": ""2022-07-01"",
   ""types"": [
        ""function""
   ],
   ""buffering"": {
        ""maxItems"": 1000,
        ""maxBytes"": 262144,
        ""timeoutMs"": 100
   },
   ""destination"": {
        ""protocol"": ""HTTP"",
        ""URI"": ""http://sandbox.localdomain:8080""
   }
}";

            using var content = new StringContent(body, Encoding.UTF8, "application/json");

            using (var response = await httpClient.PutAsync(_subscriptionUrl, content))
            {
                if (!response.IsSuccessStatusCode)
                {
                    Console.WriteLine($"[{_name}] Error response received for subscription request: {await response.Content.ReadAsStringAsync()}");
                    response.EnsureSuccessStatusCode();
                }
            }
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

        public async Task Start(ConcurrentQueue<string> queue)
        {
            using (var httpClient = new HttpClient() { Timeout = Timeout.InfiniteTimeSpan })
            {
                var id = await Register(httpClient);

                Console.WriteLine($"[{_name}] Registered extension with id = {id}");

                httpClient.DefaultRequestHeaders.Add("Lambda-Extension-Identifier", id);

                await Subscribe(httpClient);

                while (true)
                {
                    var payload = await GetNext(httpClient);

                    if(payload.EventType== EventType.SHUTDOWN)
                    {
                        Console.WriteLine($"[{_name}] Shutting down extension: {payload.ShutdownReason}");

                        await SendLogs(queue);

                        break;
                    }

                    Console.WriteLine($"[{_name}] Handling invoke from extension: {payload.RequestId}");

                    await SendLogs(queue);
                }
            }
        }

        public async Task SendLogs(ConcurrentQueue<string> queue)
        {
            using (var httpClient = new HttpClient())
            {
                while (queue.TryDequeue(out var item))
                {
                    var content = new StringContent(item, Encoding.UTF8, "application/json");

                    await httpClient.PostAsync(_targetUrl, content);
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

Modify the `Program.cs` file with the following content:

```csharp
using Microsoft.AspNetCore.Http.Json;
using System.Collections.Concurrent;
using System.Reflection;

namespace MyExtension;

class Program
{
    static async Task Main(string[] args)
    {
        var name = (1 == args.Length)
            ? args[0]
            : Assembly.GetEntryAssembly()?.GetName()?.Name;

        var builder = WebApplication.CreateBuilder(args);

        builder.Services.Configure<JsonOptions>(options =>
        {
            options.SerializerOptions.PropertyNameCaseInsensitive= true;
        });

        var queue = new ConcurrentQueue<string>();

        var app = builder.Build();

        app.Urls.Add("http://sandbox.localdomain:8080");

        app.MapPost("/", (Log[] logs) =>
        {
            foreach (var log in logs)
            {
                queue.Enqueue(log.Record);
            }
            return Results.Ok();
        });

        await app.StartAsync();

        Console.WriteLine("Web server started");

        await new Extension(name).Start(queue);
    }
}
```

The `Function.cs` file will change to log a message during the request:

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
            context.Logger.LogInformation($"log from function: {Guid.NewGuid()}");

            await Task.Delay(Random.Shared.Next(25, 100));

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

And finally, we will update the `template.yaml` with the URL where we will send the logs:

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
      Environment:
        Variables:
          TOOL_API_URL: https://webhook.site/24da577c-e3ec-44f5-9b06-ba371203a54f
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

Run `sam build` and `sam deploy --guided` and follow the instructions to deploy the Lambda function and extension. Start to send requests to the Lambda function and check the logs:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1681003800641/e44c4100-9f89-4fdf-ac01-2ee2d93a1d06.png align="center")

Eventually, the logs arrive at our [target URL](https://docs.webhook.site/):

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1681003882723/2960cac5-d1fc-4afe-94de-a2712e524988.png align="center")

The code is available [**here**](https://github.com/raulnq/aws-lambda-extensions/tree/telemetry-api). Thanks, and happy coding.