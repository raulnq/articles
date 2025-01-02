---
title: "Safe AWS Lambda Deployments with AWS SAM"
datePublished: Thu Jan 02 2025 00:55:45 GMT+0000 (Coordinated Universal Time)
cuid: cm5em7ed7000009jpgz2fevcb
slug: safe-aws-lambda-deployments-with-aws-sam
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1735652489020/e5a98bc8-de6e-4969-8f83-dd816eb84cad.png
tags: dotnet, aws-lambda, aws-sam, aws-codedeploy, canary-deployment

---

Having the right rollout strategy is crucial, depending on the criticality and needs of our applications. This applies whether we are developing a server-based or serverless application. With that in mind, AWS Lambda provides a set of features to help us achieve this goal:

* [Versions](https://docs.aws.amazon.com/lambda/latest/dg/configuration-versions.html): AWS Lambda allows us to publish immutable versions of our function, including code, runtime, architecture, memory, layers, and most other configuration settings (with exceptions like triggers, destinations, and provisioned concurrency). This provides the ability to invoke a specific version of a function at any time.
    
* [Alias](https://docs.aws.amazon.com/lambda/latest/dg/configuration-aliases.html): An alias is a named pointer to a specific version. It provides a mechanism to route traffic to one version or split traffic between two.
    

With just these two features, we can implement different deployment strategies.

## Blue/Green Deployments

The idea behind blue/green deployment is to switch traffic between two identical environments running different versions of our application.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1735593300592/11407d78-cfb3-47e4-97d4-a19ecb201e3d.png align="center")

The initial state of our environment is an AWS Lambda function version `1` with an alias named `PROD` pointing to it. Then, the AWS API Gateway uses this alias to route requests from the clients.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1735594011063/7da78fda-77b3-484c-9431-71dd60af8120.png align="center")

A new function version is deployed alongside version `1`. Allowing us to test the new version before sending real traffic.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1735594179860/466f495e-f3ef-4a18-b3c8-dcfd0b470e5a.png align="center")

Once the new version is verified, we update the alias to point to version `2`, completing the deployment.

## Canary Deployment

Under this strategy, we send a small portion of the traffic to the new version, which lets us monitor the function and roll back to the stable version if any errors occur. If no issues arise, we slowly increase the traffic until all of it is redirected to the new version.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1735599560003/53f8cd71-770b-44d2-99a7-9c05742b6ed3.png align="center")

A new function version is deployed alongside version `1`. This time, we use the [traffic shifting feature](https://docs.aws.amazon.com/lambda/latest/dg/configuring-alias-routing.html) in the alias to route traffic between the two versions. Version `2` initially receives only 10% of the traffic while we monitor it for any errors.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1735600463996/a8c9928f-bd13-4fc6-bffd-a1554cecba1d.png align="center")

After some time, we may decide to increase the traffic by 10%, repeating the process until all the traffic is routed to the new version. These two deployment strategies can be done manually, or we can use other AWS services to automate them.

## AWS CodeDeploy

> CodeDeploy is a deployment service that automates application deployments to Amazon EC2 instances, on-premises instances, serverless Lambda functions, or Amazon ECS services.

[AWS CodeDeploy](https://docs.aws.amazon.com/codedeploy/latest/userguide/welcome.html) offers features to manage traffic shifting in AWS Lambda functions. It allows us to set up AWS CloudWatch Alarms to detect issues and automatically roll back to a previous version if needed. Additionally, it provides hooks (before and after traffic shifting) to run custom logic to either continue or roll back the deployment. In this article, we won't go into detail about this service, but we believe it's important to understand the deployment workflow.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1735656859531/c9a132b6-a7e5-4d3a-b337-89871e417b65.png align="center")

As a prerequisite, AWS CodeDeploy expects a Lambda alias to be properly configured with the stable version and the new version already deployed.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1735663308803/64cba2b1-9d3f-46ed-be6f-5b1b4947a329.png align="center")

Before starting the traffic shifting, AWS CodeDeploy will invoke the PreTraffic Hook Lambda function (pre-deployment validation). This Lambda function must respond to AWS CodeDeploy with a status of success or failure. If it fails, AWS CodeDeploy will abort the deployment. If it succeeds, AWS CodeDeploy will continue with the traffic shifting.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1735665429219/0352f154-be09-4a96-b3c9-eaac184b0649.png align="center")

AWS CodeDeploy will start traffic shifting by routing a percentage of the load and gradually increasing it over time until all the traffic is moved to the new version. During traffic shifting, if any of the CloudWatch Alarms go to the `Alarm` state, AWS CodeDeploy will immediately flip the alias back to the old version.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1735667048052/54022f0e-6086-4c12-a60f-87f5e96b720d.png align="center")

After traffic shifting completes, AWS CodeDeploy will invoke the PostTraffic hook Lambda function (post-deployment validation). This is similar to the PreTraffic hook where the function must callback to CodeDeploy to report a success or a failure.

Luckily for us, everything we've seen so far can be easily implemented using AWS SAM, so let's get started.

## Pre-requisites

* Have an [IAM User](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html#id_users_create_console) with programmatic access.
    
* Install the Amazon Lambda Templates (`dotnet new -i Amazon.Lambda.Templates`)
    
* Install the Amazon Lambda Tools (`dotnet tool install -g Amazon.Lambda.Tools`)
    
* Install [AWS SAM CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html).
    

## The Backend Services

Run the following commands to set up our Lambda functions:

```powershell
dotnet new lambda.EmptyFunction -n MyLambda -o .
dotnet add src/MyLambda package AWSSDK.CodeDeploy
dotnet add src/MyLambda package Amazon.Lambda.APIGatewayEvents
dotnet add src/MyLambda package AWSSDK.Lambda
dotnet new sln -n MyApplications
dotnet sln add --in-root src/MyLambda
```

Open the `Program.cs` file and update the content as follows:

```csharp
using Amazon.CodeDeploy.Model;
using Amazon.Lambda;
using Amazon.Lambda.APIGatewayEvents;
using Amazon.Lambda.Core;
using Amazon.Lambda.Model;
using System.Text;
using Environment = System.Environment;

[assembly: LambdaSerializer(typeof(Amazon.Lambda.Serialization.SystemTextJson.DefaultLambdaJsonSerializer))]
namespace MyLambda;

public class Function
{
    private readonly AmazonLambdaClient _lambdaClient;

    public Function()
    {
        _lambdaClient = new AmazonLambdaClient();
    }

    public APIGatewayHttpApiV2ProxyResponse FunctionHandler(APIGatewayHttpApiV2ProxyRequest input, ILambdaContext context)
    {
        return new APIGatewayHttpApiV2ProxyResponse
        {
            Body = $"{{'version':'{context.FunctionVersion}'}}",
            StatusCode = 200,
            Headers = new Dictionary<string, string> { { "Content-Type", "application/json" } }
        };
    }

    public async Task PreFunctionHandler(PutLifecycleEventHookExecutionStatusRequest request, ILambdaContext context)
    {
        context.Logger.Log("Processing lifecycle pre-traffic hook");
        var status = Amazon.CodeDeploy.LifecycleEventStatus.Failed;
        var function = Environment.GetEnvironmentVariable("TARGET");
        try
        {
            context.Logger.Log($"Invoking {function}");
            var invokeRequest = new InvokeRequest
            {
                FunctionName = function,
                Payload = "{}",               
                InvocationType = InvocationType.RequestResponse
            };
            var invokeResponse = await _lambdaClient.InvokeAsync(invokeRequest);
            var payload = Encoding.UTF8.GetString(invokeResponse.Payload.ToArray());
            context.Logger.Log($"Response {payload}");
            status = Amazon.CodeDeploy.LifecycleEventStatus.Succeeded;
        }
        catch (Exception ex)
        {
            status = Amazon.CodeDeploy.LifecycleEventStatus.Failed;
            context.Logger.LogError($"Error {ex.Message}");
        }
        finally
        {
            context.Logger.Log("Calling CodeDeploy");
            using var codedeploy = new Amazon.CodeDeploy.AmazonCodeDeployClient();
            var statusRequest = new PutLifecycleEventHookExecutionStatusRequest()
            {
                DeploymentId = request.DeploymentId,
                LifecycleEventHookExecutionId = request.LifecycleEventHookExecutionId,
                Status = status,
            };
            await codedeploy.PutLifecycleEventHookExecutionStatusAsync(statusRequest).ConfigureAwait(false);
        }
    }
    public async Task PostFunctionHandler(PutLifecycleEventHookExecutionStatusRequest request, ILambdaContext context)
    {
        context.Logger.Log("Processing lifecycle post-traffic hook.");
        var status = Amazon.CodeDeploy.LifecycleEventStatus.Failed;
        var function = Environment.GetEnvironmentVariable("TARGET");
        try
        {
            context.Logger.Log($"Invoking {function}");
            var invokeRequest = new InvokeRequest
            {
                FunctionName = function,
                Payload = "{}",
                InvocationType = InvocationType.RequestResponse
            };
            var invokeResponse = await _lambdaClient.InvokeAsync(invokeRequest);
            var payload = Encoding.UTF8.GetString(invokeResponse.Payload.ToArray());
            context.Logger.Log($"Response {payload}");
            status = Amazon.CodeDeploy.LifecycleEventStatus.Succeeded;
        }
        catch (Exception ex)
        {
            status = Amazon.CodeDeploy.LifecycleEventStatus.Failed;
            context.Logger.LogError($"Error {ex.Message}");
        }
        finally
        {
            context.Logger.Log("Calling CodeDeploy");
            using var codedeploy = new Amazon.CodeDeploy.AmazonCodeDeployClient();
            var statusRequest = new PutLifecycleEventHookExecutionStatusRequest()
            {
                DeploymentId = request.DeploymentId,
                LifecycleEventHookExecutionId = request.LifecycleEventHookExecutionId,
                Status = status,
            };
            await codedeploy.PutLifecycleEventHookExecutionStatusAsync(statusRequest).ConfigureAwait(false);
        }
    }
}
```

The `FunctionHandler` function is what we will deploy, while the `PreFunctionHandler` and `PostFunctionHandler` are the hooks that will be called during the process. Both hooks follow the same logic: they get the ARN of our function, invoke it, and then notify CodeDeploy of the result.

## AWS SAM template

At the solution level, create a `template.yml` file with the following content:

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
      AutoPublishAlias: prod
      DeploymentPreference:
        Type: Linear10PercentEvery10Minutes
        Alarms:
          - !Ref MyAlarm
        Hooks:
          PreTraffic: !Ref PreTrafficLambdaFunction
          PostTraffic: !Ref PostTrafficLambdaFunction
      Events:
        get:
          Type: Api
          Properties:
            Path: /version
            Method: get

  MyAlarm:
    Type: "AWS::CloudWatch::Alarm"
    Properties:
      AlarmDescription: Lambda Function Error > 0
      ComparisonOperator: GreaterThanThreshold
      Dimensions:
        - Name: Resource
          Value: !Sub "${MyFunction}:prod"
        - Name: FunctionName
          Value: !Ref MyFunction
        - Name: ExecutedVersion
          Value: !GetAtt MyFunction.Version.Version
      EvaluationPeriods: 2
      MetricName: Errors
      Namespace: AWS/Lambda
      Period: 60
      Statistic: Sum
      Threshold: 0

  PreTrafficLambdaFunction:
    Type: AWS::Serverless::Function
    Properties:
      Timeout: 60
      MemorySize: 512
      Handler: MyLambda::MyLambda.Function::PreFunctionHandler
      CodeUri: ./src/MyLambda/
      FunctionName: 'CodeDeployHook_preTrafficHook'
      Policies:
        - Version: "2012-10-17"
          Statement:
          - Effect: "Allow"
            Action:
              - "codedeploy:PutLifecycleEventHookExecutionStatus"
            Resource:
              !Sub 'arn:${AWS::Partition}:codedeploy:${AWS::Region}:${AWS::AccountId}:deploymentgroup:${ServerlessDeploymentApplication}/*'
        - Version: "2012-10-17"
          Statement:
          - Effect: "Allow"
            Action:
              - "lambda:InvokeFunction"
            Resource: !Ref MyFunction.Version
      Runtime: dotnet8
      Architectures:
        - x86_64   
      DeploymentPreference:
        Enabled: False
      Environment:
        Variables:
          TARGET: !Ref MyFunction.Version

  PostTrafficLambdaFunction:
    Type: AWS::Serverless::Function
    Properties:
      Timeout: 60
      MemorySize: 512
      Handler: MyLambda::MyLambda.Function::PostFunctionHandler
      CodeUri: ./src/MyLambda/
      FunctionName: 'CodeDeployHook_postTrafficHook'
      Policies:
        - Version: "2012-10-17"
          Statement:
          - Effect: "Allow"
            Action:
              - "codedeploy:PutLifecycleEventHookExecutionStatus"
            Resource:
              !Sub 'arn:${AWS::Partition}:codedeploy:${AWS::Region}:${AWS::AccountId}:deploymentgroup:${ServerlessDeploymentApplication}/*'
        - Version: "2012-10-17"
          Statement:
          - Effect: "Allow"
            Action:
              - "lambda:InvokeFunction"
            Resource: !GetAtt MyFunction.Arn
      Runtime: dotnet8
      Architectures:
        - x86_64    
      DeploymentPreference:
        Enabled: False
      Environment:
        Variables:
          TARGET: !GetAtt MyFunction.Arn

Outputs:
  MyApiEndpoint:
    Description: "API endpoint"
    Value: !Sub "https://${ServerlessRestApi}.execute-api.${AWS::Region}.amazonaws.com/Prod/version"
```

In AWS SAM, the magic happens around two properties: `AutoPublishAlias` and `DeploymentPreference` in the `AWS::Serverless::Function` resource. The `AutoPublishAlias` property does the following:

* Creates an alias with the specified name.
    
* Creates and publishes a version with the latest code.
    
* Points the alias to the latest version.
    
* Points all the event sources to the alias.
    

By setting up the [`DeploymentPreference`](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/sam-property-function-deploymentpreference.html#sam-function-deploymentpreference-hooks) property, we can enable gradual deployment. Let's explore the options available to us:

* `Type`: This property lets us specify how traffic should be shifted between versions.
    
    * `Canary`: Traffic is shifted in two increments (`Canary10Percent30Minutes`, `Canary10Percent5Minutes`, `Canary10Percent10Minutes`, `Canary10Percent15Minutes`).
        
    * `Linear`: Traffic is shifted in equal increments with an equal number of minutes between each increment (`Linear10PercentEvery10Minutes`, `Linear10PercentEvery1Minute`, `Linear10PercentEvery2Minutes`, `Linear10PercentEvery3Minutes`).
        
    * `AllAtOnce`: All traffic is shifted at once.
        
* `Alarms`: During traffic shifting, if any AWS CloudWatch Alarms change to the `Alarm` state, AWS CodeDeploy will immediately switch the alias back to the old version.
    
* `Hooks`: These are pre-traffic and post-traffic functions that run checks before traffic shifting begins to the new version and after traffic shifting finishes. With either hook, we can run logic to decide if the deployment should succeed or fail and then notify AWS CodeDeploy. Remember, we need to add the proper permission to allow our functions to call AWS CodeDeploy.
    

> If the hook functions are created by the same SAM template being deployed, make sure to turn off traffic shifting deployments for the hook functions.

Behind the scenes, AWS SAM does the following:

* Creates one `AWS::CodeDeploy::Application` per stack
    
* Creates one `AWS::CodeDeploy::DeploymentGroup` per `AWS::Serverless::Function` resource.
    
* Creates one `AWS::IAM::Role` called `CodeDeployServiceRole`, if no role is provided in the `DeploymentPreference` property. This role only allows invoking functions with names that start with `CodeDeployHook_`.
    
* Add an [`UpdatePolicy`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-attribute-updatepolicy.html#cfn-attributes-updatepolicy-codedeploylambdaaliasupdate) to the alias to perform an AWS CodeDeploy deployment when the version changes.
    

Run the following commands to deploy the resources to AWS:

```powershell
sam build
sam deploy --guided
```

In this final practical example, we are not doing a pure blue/green deployment or a canary deployment; it's more of a mix. However, depending on how we set up the shifting type and use the hooks, we can mimic exactly what we explained at the beginning of this article. As a final note, don't forget to clean up old function versions. There is a limit of [75 GB](https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-limits.html) that we don't want to reach. You can find all the code [here](https://github.com/raulnq/aws-lambda-safe-deployments). Thanks, and happy coding.