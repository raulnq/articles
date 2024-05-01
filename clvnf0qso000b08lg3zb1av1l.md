---
title: "How to Secure AWS Lambda Functions Using Amazon API Gateway and AWS IAM"
datePublished: Wed May 01 2024 06:07:29 GMT+0000 (Coordinated Universal Time)
cuid: clvnf0qso000b08lg3zb1av1l
slug: how-to-secure-aws-lambda-functions-using-amazon-api-gateway-and-aws-iam
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1714425179303/968a1fde-afe3-4fc1-8d0b-ff7d38292222.png
tags: authorization, aws-lambda, aws-iam, amazon-api-gateway

---

AWS Amazon API Gateway supports multiple methods to secure our REST APIs. One of them, [AWS IAM Authorization](https://docs.aws.amazon.com/apigateway/latest/developerguide/permissions.html), is a good fit in scenarios where both the caller and the callee are within AWS, and we want to avoid sharing keys and secrets between them. In this scenario, the callee is a Lambda function behind AWS API Gateway, and the caller is an application hosted on EKS.

## Pre-requisites

* Install [Docker Desktop.](https://docs.docker.com/desktop/install/windows-install/)
    
* Enable [Kubernetes](https://docs.docker.com/desktop/kubernetes/) (the standalone version included in Docker Desktop).
    
* An [IAM User](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html#id_users_create_console) with programmatic access.
    
* Install the [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-creds).
    
* Install [Terraform CLI](https://learn.hashicorp.com/tutorials/terraform/install-cli).
    
* Install the Amazon Lambda Templates (`dotnet new -i Amazon.Lambda.Templates`)
    
* Install the Amazon Lambda Tools (`dotnet tool install -g Amazon.Lambda.Tools`)
    
* Install [AWS SAM CLI](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html).
    

## The Lambda function

Run the following commands to set up our Lambda function:

```powershell
dotnet new lambda.EmptyFunction -n MyLambda -o .
dotnet add src/MyLambda package Amazon.Lambda.APIGatewayEvents
dotnet new sln -n MyApplications
dotnet sln add --in-root src/MyLambda
```

Open the `Program.cs` file and update the content as follows:

```powershell
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

A simple Lambda function is defined. At the solution level, create a `template.yml` file with the following content:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: >
  SAM
Resources:
  MyApiFunction:
    Type: AWS::Serverless::Function
    Properties:
      MemorySize: 512
      Runtime: dotnet8
      Architectures:
        - x86_64    
      Handler: MyLambda::MyLambda.Function::FunctionHandler
      CodeUri: ./src/MyLambda/
      Events:
        ListPosts:
          Type: Api
          Properties:
            Path: /hello-world
            Method: get
            Auth:
              Authorizer: AWS_IAM

  MyRoleToAssume:
    Type: AWS::IAM::Role
    Properties:
      RoleName: MyRoleToAssume
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              AWS: '*'
            Action: sts:AssumeRole
      Policies:
        - PolicyName: MyPolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action: 'execute-api:Invoke'
                Resource: !Sub 'arn:aws:execute-api:${AWS::Region}:${AWS::AccountId}:${ServerlessRestApi}/*/GET/hello-world'
              - Effect: Allow
                Action: 'lambda:InvokeFunction'
                Resource: !GetAtt MyApiFunction.Arn
Outputs:
  MyApiEndpoint:
    Description: "API endpoint"
    Value: !Sub "https://${ServerlessRestApi}.execute-api.${AWS::Region}.amazonaws.com/Prod/hello-world"
  MyRole:
    Description: "Role arn"
    Value: !GetAtt MyRoleToAssume.Arn
```

Here, we create a standard `AWS::Serverless::Function` resource with an event source of type `Api`. The event source `Api` includes an `Auth` property that defines an `Authorizer` of type `AWS_IAM`. The `AWS::IAM::Role` resource is the role that our caller will assume to call the Lambda function through the Amazon API Gateway. Note that **we need two permissions**: one to [invoke the endpoint](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-control-access-using-iam-policies-to-invoke-api.html#api-gateway-who-can-invoke-an-api-method-using-iam-policies) and the other to invoke the function itself. Run the following commands to deploy the resources to AWS:

```powershell
sam build
sam deploy --guided --capabilities CAPABILITY_NAMED_IAM
```

When creating a named IAM resource, we must specify the `CAPABILITY_NAMED_IAM` parameter. That ends the section on the Lambda function and Amazon API Gateway. If we try to call the endpoint, we will receive a response like:

```json
{
    message: "Missing Authentication Token"
}
```

## The EKS application

As mentioned, we will host the client application in an EKS cluster. Run the following commands:

```powershell
dotnet new webapi -n MyClient -o src/MyClient
dotnet sln add --in-root src/MyClient
dotnet add src/MyClient package AWSSDK.SecurityToken
dotnet add src/MyClient package AWSSDK.Extensions.NETCore.Setup
dotnet add src/MyClient package Aws4RequestSigner
```

Open the `Program.cs` file and update the content as follows:

```powershell
using Amazon.Extensions.NETCore.Setup;
using Amazon.SecurityToken;
using Amazon.SecurityToken.Model;
using Aws4RequestSigner;
using Microsoft.AspNetCore.Mvc;
using System.Text;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();
builder.Services.AddDefaultAWSOptions(builder.Configuration.GetAWSOptions());
builder.Services.AddAWSService<IAmazonSecurityTokenService>();
var app = builder.Build();
var endpoint = builder.Configuration.GetValue<string>("AWS_LAMBDA_ENDPOINT")!;
var role = builder.Configuration.GetValue<string>("AWS_ROLE_TO_ASSUME")!;
var region = builder.Configuration.GetValue<string>("AWS_REGION")!;
// Configure the HTTP request pipeline.
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();

app.MapGet("/proxy", async ([FromServices] IAmazonSecurityTokenService stsClient) =>
{
    var assumeRoleRequest = new AssumeRoleRequest
    {
        RoleArn = role,
        RoleSessionName = Guid.NewGuid().ToString(),
        DurationSeconds = 3600
    };

    var assumeRoleResponse = await stsClient.AssumeRoleAsync(assumeRoleRequest);
    var credentials = assumeRoleResponse.Credentials;

    var signer = new AWS4RequestSigner(credentials.AccessKeyId, credentials.SecretAccessKey);
    var content = new StringContent(string.Empty, Encoding.UTF8, "application/json");
    var request = new HttpRequestMessage
    {
        Method = HttpMethod.Get,
        RequestUri = new Uri(endpoint),
        Content = content
    };
    request.Headers.TryAddWithoutValidation("X-Amz-Security-Token", credentials.SessionToken);
    request = await signer.Sign(request, "execute-api", region);
    var client = new HttpClient();
    var response = await client.SendAsync(request);
    return await response.Content.ReadAsStringAsync();
})
.WithOpenApi();
app.Run();
```

We will assume the role created by AWS SAM with the `IAmazonSecurityTokenService` interface and use the response (`AccessKey`, `SecretAccessKey`, and `SessionToken`) to [sign](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_aws-signing.html) the request against Amazon API Gateway. To sign the request, we are using an unofficial library named [aws-signer-v4-dot-net](https://github.com/tsibelman/aws-signer-v4-dot-net). Another available option is the [AwsSignatureVersion4](https://github.com/FantasticFiasco/aws-signature-version-4) library. Note that this is a basic implementation; the token should be cached somewhere to be used until it is valid.

## Terraform

We will use [Terraform](https://developer.hashicorp.com/terraform?product_intent=terraform) to set up the required resources (ECR Repository and IAM Role) needed to run our application on the EKS cluster. At the project level, create a `main.tf` file with the following content:

```dockerfile
terraform {
  required_providers {
    aws = {
      source = "hashicorp/aws"
      version = "5.31.0"
    }
  }
  backend "local" {}
}

provider "aws" {
  region      = "<MY_REGION>"
  profile     = "<MY_AWS_PROFILE>"
  max_retries = 2
}

locals {
  repository_name         = "my-client-repository"
  cluster_name            = "<MY_K8S_CLUSTER_NAME>"
  role_name               = "my-role"
  namespace               = "<MY_K8S_NAMESPASE>"
  policy_name			  = "my-policy"
}

resource "aws_ecr_repository" "repository" {
  name                 = local.repository_name
  image_tag_mutability = "MUTABLE"
  image_scanning_configuration {
    scan_on_push = false
  }
}

data "aws_iam_policy_document" "my_policy_document" {
  statement {
    effect    = "Allow"
    actions = [
      "s3:PutObject"
    ]
    resources = [
      "*"
    ]
  }
}

resource "aws_iam_policy" "my_policy" {
  name   = local.policy_name
  path   = "/"
  policy = data.aws_iam_policy_document.my_policy_document.json
}

data "aws_eks_cluster" "cluster" {
  name = local.cluster_name
}

module "iam_assumable_role_with_oidc" {
  source                       = "terraform-aws-modules/iam/aws//modules/iam-assumable-role-with-oidc"
  version                      = "4.14.0"
  oidc_subjects_with_wildcards = ["system:serviceaccount:${local.namespace}:*"]
  create_role                  = true
  role_name                    = local.role_name
  provider_url                 = data.aws_eks_cluster.cluster.identity[0].oidc[0].issuer
  role_policy_arns = [
    aws_iam_policy.my_policy.arn,
  ]
  number_of_role_policy_arns = 1
}

output "role_arn" {
  value = module.iam_assumable_role_with_oidc.iam_role_arn
}

output "repository_url" {
  value = aws_ecr_repository.repository.repository_url
}
```

Notice that the action `s3:PutObject` is not necessary. It's just a placeholder to include something in the policy. Run the following commands to create the resources in AWS:

```powershell
terraform init
terraform plan -out app.tfplan
terraform apply 'app.tfplan'
```

## Docker Image

At the client project level, create a `Dockerfile` with the following content:

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
COPY ["src/MyClient/MyClient.csproj", "MyClient/"]
RUN dotnet restore "MyClient/MyClient.csproj"
COPY src/ .
WORKDIR "/MyClient"
RUN dotnet build "MyClient.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "MyClient.csproj" -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "MyClient.dll"]
```

Run the following command at the solution level to upload the image to the Amazon ECR repository:

```powershell
aws ecr get-login-password --region <MY_REGION> --profile <MY_AWS_PROFILE> | docker login --username AWS --password-stdin <MY_ACCOUNT_ID>.dkr.ecr.<MY_REGION>.amazonaws.com
docker build -t <MY_ACCOUNT_ID>.dkr.ecr.<MY_REGION>.amazonaws.com/my-client-repositoy:1.0 -f .\src\MyClient\Dockerfile .
docker push <MY_ACCOUNT_ID>.dkr.ecr.<MY_REGION>.amazonaws.com/my-client-repositoy:1.0
```

## Kubernetes

We will deploy our Kubernetes resources, including the service account, deployment, and service, to the EKS cluster. Create an `eks.yaml` file with the following content:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
 name: myclient-sa
 annotations:
   eks.amazonaws.com/role-arn: arn:aws:iam::<MY_ACCOUNT_ID>:role/my-role
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myclient-deployment
  labels:
    app: myclient
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myclient
  template:
    metadata:
      labels:
        app: myclient
    spec:
      serviceAccountName: myclient-sa
      containers:
        - name: api-container
          env:
            - name: ASPNETCORE_ENVIRONMENT
              value: Development
            - name: ASPNETCORE_HTTP_PORTS
              value: '80'
            - name: AWS_LAMBDA_ENDPOINT
              value: 'https://<MY_API_GATEWAY_ID>.execute-api.<MY_REGION>.amazonaws.com/Prod/hello-world'
            - name: AWS_ROLE_TO_ASSUME
              value: 'arn:aws:iam::<MY_ACCOUNT_ID>:role/MyRoleToAssume'
          image: <MY_ACCOUNT_ID>.dkr.ecr.<MY_REGION>.my-client-repository:1.0
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
          resources:
            limits:
              cpu: 500m
              memory: 500Mi
            requests:
              cpu: 250m
              memory: 250Mi
---
apiVersion: v1
kind: Service
metadata:
  name: myclient-service
  labels:
    app: myclient
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: http
      protocol: TCP
      name: http
  selector:
    app: myclient
```

The IAM Role created with the Terraform script is used in the service account definition. Additionally, the IAM Role to assume and the Amazon API Gateway endpoint created by the SAM script are set up as environment variables in our client application. Run the following command to deploy the resources to the cluster:

```powershell
kubectl apply -f eks.yaml --namespace=<MY_K8S_NAMESPASE>
```

Run `kubectl get services -n <MY_K8S_NAMESPASE>` command to get our service `URL` and test the client application. We should see a response like:

```json
{
    Message: "Hello World"
}
```

You can find the code and scripts [here](https://github.com/raulnq/api-gateway-iam-authorization). Thank you, and happy coding.