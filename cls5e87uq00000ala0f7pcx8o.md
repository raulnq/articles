---
title: "Scaling Amazon Elastic Kubernetes Service Workloads with KEDA and SQS"
datePublished: Sat Feb 03 2024 01:26:20 GMT+0000 (Coordinated Universal Time)
cuid: cls5e87uq00000ala0f7pcx8o
slug: scaling-amazon-elastic-kubernetes-service-workloads-with-keda-and-sqs
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1706918624578/c36b9fe9-5b40-497a-9d86-d972362de2a5.png
tags: aws, sqs, aws-sqs, eks, keda

---

In this article, we'll explore another practical use case of [KEDA](https://keda.sh/docs/2.13/), where we scale an application dynamically in response to message traffic. This involves leveraging the combined power of Amazon EKS Cluster, KEDA, and the AWS SQS.

## Pre-requisites

* Install [Docker Desktop](https://docs.docker.com/desktop/install/windows-install/).
    
* Enable [Kubernetes](https://docs.docker.com/desktop/kubernetes/) (the standalone version included in Docker Desktop).
    
* An [Amazon EKS Cluster](https://docs.aws.amazon.com/eks/latest/userguide/clusters.html) with [KEDA](https://keda.sh/docs/2.12/) installed is required. We will use the option `operator` as the `identityOwner` for our [AWS SQS](https://keda.sh/docs/2.13/scalers/aws-sqs/) scaler. Therefore, we must grant the KEDA operator the necessary IAM permissions to access SQS. You can find an example of how to accomplish this [here](https://dev.to/carlocolumna/keda-in-amazon-eks-part-2-scale-based-on-aws-sqs-queue-5g4d).
    
* Docker Desktop Kubernetes context configured to work with the Amazon EKS cluster.
    
* An [IAM User](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html#id_users_create_console) with programmatic access.
    
* Install the [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-creds).
    
* Install [Terraform CLI](https://learn.hashicorp.com/tutorials/terraform/install-cli).
    

## The Application

Run the following commands to set up the solution:

```powershell
dotnet new web -o MyApi
dotnet new sln -n MyApi
dotnet sln add --in-root MyApi
dotnet add MyApi package AWSSDK.SQS
dotnet add MyApi package AWSSDK.Extensions.NETCore.Setup
dotnet add MyApi package AWSSDK.SecurityToken
```

We will create a background service to read messages from a queue. Create a `SqsBackgroundService.cs` file with the following content:

```csharp
using Amazon.SQS.Model;
using Amazon.SQS;
namespace MyApi;

public class SqsBackgroundService : BackgroundService
{
    private readonly string _queueUrl;
    private readonly IAmazonSQS _sqsClient;

    public SqsBackgroundService(IConfiguration configuration, IAmazonSQS sqsClient)
    {
        _sqsClient = sqsClient;
        _queueUrl = configuration.GetValue<string>("QueueUrl");
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        Console.WriteLine($"Starting polling queue at {_queueUrl}");
        while (!stoppingToken.IsCancellationRequested)
        {
            var request = new ReceiveMessageRequest
            {
                QueueUrl = _queueUrl,
                MaxNumberOfMessages = 10,
                WaitTimeSeconds = 5
            };
            var response = await _sqsClient.ReceiveMessageAsync(request);
            if (response.Messages.Any())
            {
                foreach (var msg in response.Messages)
                {
                    await Task.Delay(Random.Shared.Next(1000, 2500));
                    Console.WriteLine($"Message received with body {msg.Body}");
                    await _sqsClient.DeleteMessageAsync(new DeleteMessageRequest
                    {
                        QueueUrl = _queueUrl,
                        ReceiptHandle = msg.ReceiptHandle
                    });
                }
            }
            else
            {
                Console.WriteLine("No message available");
                await Task.Delay(TimeSpan.FromSeconds(5), stoppingToken);
            }
        }
    }
}
```

We're introducing a random delay to simulate processing time. Next, replace the contents of the `Program.cs` file with:

```csharp
using Amazon.SQS;
using Amazon.SQS.Model;
using Microsoft.AspNetCore.Mvc;
using MyApi;

var builder = WebApplication.CreateBuilder(args);

builder.Configuration
    .AddJsonFile("appsettings.json")
    .AddEnvironmentVariables();

builder.Services.AddDefaultAWSOptions(builder.Configuration.GetAWSOptions());
builder.Services.AddHostedService<SqsBackgroundService>();
builder.Services.AddAWSService<IAmazonSQS>();

var queue = builder.Configuration.GetValue<string>("QueueUrl");

var app = builder.Build();

app.MapGet("/", () => "Hello World!");

app.MapPost("/send", async ([FromServices]IAmazonSQS sqsClient, [FromBody] Request request) =>
{
    for (int i = 0; i < request.Messages; i++)
    {
        var body = $"{Guid.NewGuid()}";
        var message = new SendMessageRequest(queue, body);
        await sqsClient.SendMessageAsync(message);
        Console.WriteLine($"Message sent with body {body}");
    }
});

app.Run();

public record Request(int Messages);
```

Here, we configure all the code to register the `IAmazonSQS` class and set up an endpoint for sending messages to a queue.

## Terraform

We will use [Terraform](https://developer.hashicorp.com/terraform?product_intent=terraform) to create the necessary resources for running our application on the Kubernetes cluster. At the project level, create a `main.tf` file with the following content:

```json
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
  repository_name         = "myrepo"
  cluster_name            = "<MY_K8S_CLUSTER_NAME>"
  role_name               = "myrole"
  namespace               = "<MY_K8S_NAMESPASE>"
}

resource "aws_ecr_repository" "repository" {
  name                 = local.repository_name
  image_tag_mutability = "MUTABLE"

  image_scanning_configuration {
    scan_on_push = false
  }
}

data "aws_iam_policy" "sqs_policy" {
  name = "AmazonSQSFullAccess"
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
    data.aws_iam_policy.sqs_policy.arn,
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

We are creating an [Amazon ECR](https://docs.aws.amazon.com/AmazonECR/latest/userguide/Repositories.html) repository to upload our application's image and an [IAM Role](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html) for our Pod with sufficient permissions for AWS SQS. Run the following commands to create the resources in AWS:

```powershell
terraform init
terraform plan -out app.tfplan
terraform apply 'app.tfplan'
```

## Docker Image

Create a `Dockerfile` with the following content:

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
COPY ["MyApi/MyApi.csproj", "MyApi/"]
RUN dotnet restore "MyApi/MyApi.csproj"
COPY . .
WORKDIR "/MyApi"
RUN dotnet build "MyApi.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "MyApi.csproj" -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "MyApi.dll"]
```

Run the following command to upload the image to the Amazon ECR repository:

```powershell
aws ecr get-login-password --region <MY_REGION> --profile <MY_AWS_PROFILE> | docker login --username AWS --password-stdin <MY_ACCOUNT_ID>.dkr.ecr.<MY_REGION>.amazonaws.com
docker build -t <MY_ACCOUNT_ID>.dkr.ecr.<MY_REGION>.amazonaws.com/myrepo:1.0 -f .\MyApi\Dockerfile .
docker push <MY_ACCOUNT_ID>.dkr.ecr.<MY_REGION>.amazonaws.com/myrepo:1.0
```

## Kubernetes

To interact with the AWS SQS API, we must assume an AWS IAM Role from our Pod through a Service Account. You can find more information [here](https://blog.raulnq.com/how-to-assume-an-aws-iam-role-from-an-eks-pod). Create a `serviceaccount.yaml` file with the following content:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
 name: kead-sa
 annotations:
   eks.amazonaws.com/role-arn: arn:aws:iam::<MY_ACCOUNT_ID>:role/myrole
```

Next, we will use the Service Account mentioned earlier. Create a `deployment.yaml` file with the following content:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: keda-deployment
  labels:
    app: api
spec:
  replicas: 1
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      serviceAccountName: kead-sa
      containers:
        - name: container
          env:
            - name: ASPNETCORE_ENVIRONMENT
              value: Development
            - name: QueueUrl
              value: <MY_QUEUE_URL>
          image: <MY_ACCOUNT_ID>.dkr.ecr.<MY_REGION>.amazonaws.com/myrepo:1.0
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
```

Our application will be accessible through a Service using a Load Balancer as its type. Create a `service.yaml` file containing the following content:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: keda-service
  labels:
    app: api
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: 8080
      protocol: TCP
      name: http
  selector:
    app: api
```

Finally, create a [Scaled Object](https://keda.sh/docs/2.13/scalers/aws-sqs/) referencing the Deployment created earlier. Create a `scaledobject.yaml` file containing the following content:

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: keda-so
spec:
  minReplicaCount: 1
  maxReplicaCount: 15  
  scaleTargetRef:
    name: keda-deployment
  triggers:
  - type: aws-sqs-queue
    metadata:
      queueURL: <MY_QUEUE_URL>
      queueLength: "5"
      awsRegion: <MY_REGION>
      identityOwner: operator
```

Execute the following commands to deploy the application to the cluster:

```powershell
kubectl apply -f serviceaccount.yaml --namespace=<MY_K8S_NAMESPASE>
kubectl apply -f deployment.yaml --namespace=<MY_K8S_NAMESPASE>
kubectl apply -f service.yaml --namespace=<MY_K8S_NAMESPASE>
kubectl apply -f scaledobject.yaml --namespace=<MY_K8S_NAMESPASE>
```

Run `kubectl get services --namespace=<MY_K8S_NAMESPACE>` to see the URL assigned to our application. The output will look something like this:

```powershell
NAME           TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kead-service   LoadBalancer   10.100.67.26    <MY_URL>      80:32349/TCP   27m
```

## **Tests**

Send a POST request to the previous URL with the following payload:

```json
{
    "Messages":1000
}
```

Next, we can observe how the Deployment scales up to create more pods in response to the load on the application. Run `kubectl get pods --namespace=<MY_K8S_NAMESPACE>`:

```json
NAME                               READY   STATUS    RESTARTS   AGE
keda-deployment-65466c4d89-2dtmq   1/1     Running   0          19s
keda-deployment-65466c4d89-47tnc   1/1     Running   0          4s
keda-deployment-65466c4d89-5n6s7   1/1     Running   0          19s
keda-deployment-65466c4d89-5rdcs   1/1     Running   0          4s
keda-deployment-65466c4d89-6crln   1/1     Running   0          34s
keda-deployment-65466c4d89-9vxkx   1/1     Running   0          34s
keda-deployment-65466c4d89-d6kkp   1/1     Running   0          4s
keda-deployment-65466c4d89-dkjwc   1/1     Running   0          19s
keda-deployment-65466c4d89-g22vm   1/1     Running   0          19s
keda-deployment-65466c4d89-hgx9z   1/1     Running   0          4s
keda-deployment-65466c4d89-nkc2n   1/1     Running   0          4s
keda-deployment-65466c4d89-p2nzc   1/1     Running   0          4s
keda-deployment-65466c4d89-w4mhz   1/1     Running   0          4s
keda-deployment-65466c4d89-wz6z6   1/1     Running   0          4h31m
keda-deployment-65466c4d89-xrbdj   1/1     Running   0          34s
```

You can find the code and scripts [here](https://github.com/raulnq/keda-sqs). Thank you, and happy coding.