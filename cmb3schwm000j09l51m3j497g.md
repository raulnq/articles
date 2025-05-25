---
title: "AWS Lambda SnapStart for .NET"
datePublished: Sun May 25 2025 15:00:26 GMT+0000 (Coordinated Universal Time)
cuid: cmb3schwm000j09l51m3j497g
slug: aws-lambda-snapstart-for-net
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1748127814476/10e0a934-7ff0-491d-8866-dddfd98bada4.png
tags: aws, net, dotnet, serverless, aws-lambda

---

Cold starts have long been a pain point for serverless applications, especially for latency-sensitive workloads or those built with runtimes that traditionally have longer initialization times, like .NET. [AWS Lambda SnapStart](https://docs.aws.amazon.com/lambda/latest/dg/snapstart.html) is a feature designed to significantly reduce these startup latencies, at least in certain situations (as we will see).

## How Does It Work?

SnapStart initializes our function once we publish it, then takes a snapshot of the initialized execution environment, including the loaded code, memory, and disk state, and caches it. When our function is invoked and needs to scale, AWS Lambda starts new execution environments from this cached snapshot instead of initializing them from scratch. This means that the time-consuming parts of a cold start are mostly eliminated for later invocations. The process can be broken down into the following phases:

### **Initialization**

* When we publish a new [**version**](https://docs.aws.amazon.com/lambda/latest/dg/configuration-versions.html) of a function, AWS Lambda provisions an execution environment.
    
* AWS Lambda runs our function's initialization code, which is the code outside our main handler method.
    

### **Snapshotting**

* After the initialization code is complete, AWS Lambda takes a snapshot of the entire execution environment's memory and disk state.
    
* This snapshot is encrypted and stored in a cache.
    

### Restore

* When an invocation occurs, AWS Lambda fetches the snapshot from the cache and resumes the execution environment.
    

## Runtime Hooks

When using SnapStart, AWS Lambda provides [runtime hooks](https://docs.aws.amazon.com/lambda/latest/dg/snapstart-runtime-hooks.html) that allow you to execute specific code at two points: just before the snapshot is taken and right after an execution environment is restored from a snapshot.

### Before Snapshot

The purpose of the hook is to prepare the environment for snapshotting. This is our chance to clean up or change any state that shouldn't be included in the snapshot. Common use cases include:

* Closing network connections.
    
* Release resources.
    

### After Restore

The purpose of the hook is to reinitialize or refresh temporary states that were either not included in the snapshot or need to be unique and current for the newly restored environment. Common use cases include:

* Re-establishing network connections.
    
* Refreshing configurations.
    

Generating unique identifiers.

### Considerations

* **Execution order**: If multiple handlers are registered for the same hook type, they run in the order they were registered.
    
* **Idempotency:** Hooks should ideally be idempotent.
    
* **Error Handling:** Unhandled exceptions in runtime hooks can cause the snapshot creation or the invocation to fail.
    
* **.NET Library**: Runtime hooks are available in the [Amazon.Lambda.Core](https://www.nuget.org/packages/Amazon.Lambda.Core) package. This library provides two static methods that we can use to register our runtime hooks:
    
    * `Amazon.Lambda.Core.SnapshotRestore.RegisterBeforeSnapshot.RegisterBeforeSnapshot()`
        
    * `Amazon.Lambda.Core.SnapshotRestore.RegisterBeforeSnapshot.RegisterAfterRestore()`
        
* **Registration:** Register the hooks in the function's initialization code.
    
* Check [these](https://docs.aws.amazon.com/lambda/latest/dg/snapstart-best-practices.html) best practices.
    

## Costs

We are charged for two things:

* Caching a snapshot while our function version is active.
    
* Each time, AWS Lambda resumes an execution environment by restoring our snapshot.
    

The pricing depends on the amount of memory allocated to our function. Currently, the cache costs `$0.0000015046` per GB-second stored, and the restoration costs `$0.0001397998` for each GB restored.

> Delete unused function versions because they continue to incur costs.

## When to Use?

AWS Lambda SnapStart is particularly useful in certain situations:

### Functions with Heavy Initialization

.NET functions that use frameworks like ASP.NET Core, large Dependency Injection (DI) containers, Object-Relational Mappers (ORMs) like Entity Framework Core, or load many assemblies and dependencies during startup. These all contribute to longer initialization times. SnapStart handles the heavy setup once during version publication. After that, any following calls use a cached environment, skipping these time-consuming setup steps.

### Functions with Complex Initialization

.NET functions that perform significant computation, data loading, or cache warming in their constructors or initial setup phase (outside the handler) can benefit from this. This work can be done once during the snapshot creation. Restored invocations immediately benefit from this pre-computed state without re-running the costly initialization, as long as the data is suitable for snapshotting.

## Coding Time

Let's create a simple .NET 8 Lambda function, deploy it without SnapStart, observe its cold start, then enable SnapStart and see the difference.

### Pre-requisites

* Have an IAM User with programmatic access.
    
* Install the Amazon Lambda Templates (`dotnet new -i Amazon.Lambda.Templates`)
    
* Install the Amazon Lambda Tools (`dotnet tool install -g` [`Amazon.Lambda.Tools`](http://amazon.lambda.tools/))
    
* Install [AWS SAM C](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html)[LI.](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html#id_users_create_console)
    
* Install [K6](https://grafana.com/docs/k6/latest/set-up/install-k6/).
    

### Lambda Function Code

Run the following commands to set up our Lambda function:

```powershell
dotnet new lambda.EmptyFunction -n MyLambda -o .
dotnet add src/MyLambda package Amazon.Lambda.APIGatewayEvents
```

Open the `Program.cs` file and update the content as follows:

```csharp
using Amazon.Lambda.Core;
using Amazon.Lambda.APIGatewayEvents;

// Assembly attribute to enable the Lambda function's JSON input to be converted into a .NET class.
[assembly: LambdaSerializer(typeof(Amazon.Lambda.Serialization.SystemTextJson.DefaultLambdaJsonSerializer))]

namespace MyLambda;

public class Function
{
    private Guid _identifier;

    public Function()
    {
        _identifier = Guid.NewGuid();
        var random = new Random();
        Thread.Sleep(random.Next(2000, 5000));
    }

    public APIGatewayHttpApiV2ProxyResponse FunctionHandler(APIGatewayHttpApiV2ProxyRequest request, ILambdaContext context)
    {
        context.Logger.LogInformation($"Executing function with identifier: {_identifier}");

        return new APIGatewayHttpApiV2ProxyResponse
        {
            Body = @"{""Message"":""Hello World""}",
            StatusCode = 200,
            Headers = new Dictionary<string, string> { { "Content-Type", "application/json" } }
        };
    }
}
```

Not much to see here, just simulating a heavy initialization in the function's constructor.

### AWS SAM template

Create a `template.yaml` file with the following content:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: >
  SAM

Resources:

  MyFunction:
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
        get:
          Type: Api
          Properties:
            Path: /
            Method: get

Outputs:
  MyApiEndpoint:
    Description: "API endpoint"
    Value: !Sub "https://${ServerlessRestApi}.execute-api.${AWS::Region}.amazonaws.com/Prod"
```

Run the following commands to deploy the resources to AWS:

```powershell
sam build
sam deploy --guided
```

### Test

Create a `load.js` file with the following content:

```javascript
import http from 'k6/http';
import { sleep } from 'k6';
export const options = {
  vus: 20,
  duration: '60s',
};
export default function () {
  http.get('<MY_API_ENDPOINT>');
  sleep(1);
}
```

In the script above, we invoke the endpoint with 20 virtual users for 60 seconds. Running the command `k6 run load.js`, we obtained the following results:

| avg | min | max | med | P(90) | P(95) |
| --- | --- | --- | --- | --- | --- |
| 333.9ms | 133.33ms | 6.04s | 146.55ms | 382.46ms | 395.97ms |

### **Enabling SnapStart**

Update the `template.yaml` file with the following content:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: >
  SAM

Resources:

  MyFunction:
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
      SnapStart:
        ApplyOn: PublishedVersions
      AutoPublishAlias: live
      Events:
        get:
          Type: Api
          Properties:
            Path: /
            Method: get

Outputs:
  MyApiEndpoint:
    Description: "API endpoint"
    Value: !Sub "https://${ServerlessRestApi}.execute-api.${AWS::Region}.amazonaws.com/Prod"
```

We added the `SnapStart` property. The `AutoPublishAlias` property is necessary because SnapStart applies only to published versions, not to the `$LATEST` version. This will create an alias that always points to the latest published version. Redeploy the template and run the test again.

| avg | min | max | med | P(90) | P(95) |
| --- | --- | --- | --- | --- | --- |
| 278.86ms | 133.71ms | 1.4s | 164.42ms | 383.92ms | 407.45ms |

By comparing both results, we can see a significant difference in the `max` time, which drops from `6.04` seconds to `1.4` seconds. This is mainly because the restoration time is consistent each time (second image) and takes less time than the initialization code (first image) we used.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1748181944043/2f27a728-0f1c-4683-b7ec-c8f0ceee82c1.png align="center")

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1748181968332/67c1eeda-ac32-402a-b85f-c6e88888128b.png align="center")

So that's another tool (when applicable) that can help us reduce the annoying cold start in our AWS Lambda function. You can find all the code [here](https://github.com/raulnq/aws-lambda-snapstart). Thanks, and happy coding.