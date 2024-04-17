---
title: "Scaling Amazon Elastic Kubernetes Service Workloads with KEDA, OTEL Collector,  and Amazon CloudWatch"
datePublished: Wed Apr 17 2024 01:02:38 GMT+0000 (Coordinated Universal Time)
cuid: clv33yrr7000107l65n1eglyc
slug: scaling-amazon-elastic-kubernetes-service-workloads-with-keda-otel-collector-and-amazon-cloudwatch
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1712880738094/1923f39a-aa4e-44d9-8dcc-9def5eea94f1.png
tags: net, eks, aws-cloudwatch, opentelemetry, keda

---

In our article, [Scaling Amazon Elastic Kubernetes Service Workloads with KEDA and Amazon CloudWatch](https://blog.raulnq.com/scaling-amazon-elastic-kubernetes-service-workloads-with-keda-and-amazon-cloudwatch), we suggested using the [OTEL Collector](https://opentelemetry.io/docs/collector/) to send application metrics to CloudWatch. Today, we will show how to do this in two different ways:

* Deploying the OTEL Collector as a **sidecar**, which means putting it in a container next to our application, allows us to scale and manage it separately according to workload needs. However, this method does add extra resource demands, such as CPU and memory.
    
* Deploying the OTEL Collector as an **independent service (gateway)** simplifies managing configurations and ensures consistency across multiple services. However, it can become a single point of failure for collecting telemetry data.
    

## Pre-requisites

* Install [Docker Desktop](https://docs.docker.com/desktop/install/windows-install/).
    
* Enable [Kubernetes](https://docs.docker.com/desktop/kubernetes/) (the standalone version included in Docker Desktop).
    
* An [Amazon EKS Cluster](https://docs.aws.amazon.com/eks/latest/userguide/clusters.html) with [KEDA](https://keda.sh/docs/2.12/) installed is required. We will use the option `operator` as the `identityOwner` for our [AWS CloudWatch](https://keda.sh/docs/2.12/scalers/aws-cloudwatch/) scaler. Therefore, we must grant the KEDA operator the necessary IAM permissions to access CloudWatch. You can find an example of how to accomplish this [here](https://dev.to/carlocolumna/keda-in-amazon-eks-part-2-scale-based-on-aws-sqs-queue-5g4d).
    
* Docker Desktop Kubernetes context configured to work with the Amazon EKS cluster.
    
* An [IAM User](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html#id_users_create_console) with programmatic access.
    
* Install the [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-creds).
    
* Install [Terraform CLI](https://learn.hashicorp.com/tutorials/terraform/install-cli).
    
* Install [K6](https://k6.io/docs/get-started/installation/).
    

## The Application

Run the following commands to set up the solution:

```powershell
dotnet new webapi -n MyWebApi
dotnet new sln -n MyWebApi
dotnet sln add --in-root MyApi
dotnet add MyWebApi package OpenTelemetry.Exporter.OpenTelemetryProtocol
dotnet add MyWebApi package OpenTelemetry.Instrumentation.AspNetCore
dotnet add MyWebApi package OpenTelemetry.Extensions.Hosting
```

Open the `Program.cs` file and update the content as follows:

```csharp
using OpenTelemetry.Metrics;
using OpenTelemetry.Resources;

var builder = WebApplication.CreateBuilder(args);

// Add services to the container.
// Learn more about configuring Swagger/OpenAPI at https://aka.ms/aspnetcore/swashbuckle
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

builder.Services.AddHealthChecks();
builder.Services.AddOpenTelemetry()
    .ConfigureResource(resource => resource
        .AddService(serviceName: builder.Environment.ApplicationName))
    .WithMetrics(metrics => metrics
        .AddMeter("Microsoft.AspNetCore.Hosting")
        .AddOtlpExporter());

var app = builder.Build();

// Configure the HTTP request pipeline.
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();

var summaries = new[]
{
    "Freezing", "Bracing", "Chilly", "Cool", "Mild", "Warm", "Balmy", "Hot", "Sweltering", "Scorching"
};

app.MapGet("/weatherforecast", async () =>
{
    var delay = Random.Shared.Next(0, 1500);
    await Task.Delay(delay);
    Console.WriteLine($"New request {DateTimeOffset.UtcNow} with delay of {delay} ms");
    var forecast =  Enumerable.Range(1, 5).Select(index =>
        new WeatherForecast
        (
            DateOnly.FromDateTime(DateTime.Now.AddDays(index)),
            Random.Shared.Next(-20, 55),
            summaries[Random.Shared.Next(summaries.Length)]
        ))
        .ToArray();
    return forecast;
})
.WithName("GetWeatherForecast")
.WithOpenApi();

app.Run();

record WeatherForecast(DateOnly Date, int TemperatureC, string? Summary)
{
    public int TemperatureF => 32 + (int)(TemperatureC / 0.5556);
}
```

The `AddOpenTelemetry` method provides a builder class to set up the [Open Telemetry](https://opentelemetry.io/docs/languages/net/getting-started/) standard in our application. In this example, we're using the new [ASP.NET Core metrics](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/built-in-metrics-aspnetcore) found in the `Microsoft.AspNetCore.Hosting` namespace, specifically the `http.server.active_requests` metric. The `http.server.active_requests` metric represents the number of active HTTP requests currently being handled by the application. By tracking this metric, we can identify periods of high traffic and make informed decisions about resource allocation and scaling strategies.

## Infrastructure

We will use [Terraform](https://developer.hashicorp.com/terraform?product_intent=terraform) to create the necessary resources for running our application on the Kubernetes cluster. At the project level, create a [`main.tf`](http://main.tf) file with the following content:

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
  repository_name         = "myrepository"
  cluster_name            = "<MY_K8S_CLUSTER_NAME>"
  role_name               = "myrole"
  namespace               = "<MY_K8S_NAMESPASE>"
  policy_name			  = "mypolicy"
}

resource "aws_ecr_repository" "repository" {
  name                 = local.repository_name
  image_tag_mutability = "MUTABLE"

  image_scanning_configuration {
    scan_on_push = false
  }
}

data "aws_iam_policy_document" "otel_policy_document" {
  statement {
    effect    = "Allow"
    actions = [
      "s3:ListAllMyBuckets",
      "s3:GetBucketLocation",
	  "xray:GetSamplingStatisticSummaries",
	  "logs:CreateLogStream",
	  "xray:PutTelemetryRecords",
	  "logs:DescribeLogGroups",
	  "logs:DescribeLogStreams",
	  "xray:GetSamplingRules",
	  "ssm:GetParameters",
	  "xray:GetSamplingTargets",
	  "logs:CreateLogGroup",
	  "logs:PutLogEvents",
	  "xray:PutTraceSegments"
    ]

    resources = [
      "*"
    ]
  }
}

resource "aws_iam_policy" "otel_policy" {
  name   = local.policy_name
  path   = "/"
  policy = data.aws_iam_policy_document.otel_policy_document.json
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
    aws_iam_policy.otel_policy.arn
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

We are creating an [Amazon ECR](https://docs.aws.amazon.com/AmazonECR/latest/userguide/Repositories.html) repository to upload our application's image and an [IAM Role](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html) for our Pod with enoght permissions. Run the following commands to create the resources in AWS:

```powershell
terraform init
terraform plan -out app.tfplan
terraform apply 'app.tfplan'
```

## Docker Image

At project level, create a `Dockerfile` with the following content:

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
COPY ["MyWebApi/MyWebApi.csproj", "MyWebApi/"]
RUN dotnet restore "MyWebApi/MyWebApi.csproj"
COPY . .
WORKDIR "/MyWebApi"
RUN dotnet build "MyWebApi.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "MyWebApi.csproj" -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "MyWebApi.dll"]
```

Run the following command at solution level to upload the image to the Amazon ECR repository:

```powershell
aws ecr get-login-password --region <MY_REGION> --profile <MY_AWS_PROFILE> | docker login --username AWS --password-stdin <MY_ACCOUNT_ID>.dkr.ecr.<MY_REGION>.amazonaws.com
docker build -t <MY_ACCOUNT_ID>.dkr.ecr.<MY_REGION>.amazonaws.com/myrepository:1.0 -f .\MyWebApi\Dockerfile .
docker push <MY_ACCOUNT_ID>.dkr.ecr.<MY_REGION>.amazonaws.com/myrepository:1.0
```

## **Kubernetes**

In this section we will create all the resources needed in our Kubernetes cluster using either the sidecar or the gateway approach. Choose one method, and before proceeding to the next step, delete the resources using the following command:

```powershell
kubectl delete -f <sidecar.yaml|gateway.yam> --namespace=<MY_K8S_NAMESPASE>
```

### Sidecar

Create a `sidecar.yaml` file with the following content:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
 name: mywebapi-sa
 annotations:
   eks.amazonaws.com/role-arn: arn:aws:iam::<MY_ACCOUNT_ID>:role/myrole
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mywebapi-deployment
  labels:
    app: mywebapi
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mywebapi
  template:
    metadata:
      labels:
        app: mywebapi
    spec:
      serviceAccountName: mywebapi-sa
      containers:
        - name: api-container
          env:
            - name: ASPNETCORE_ENVIRONMENT
              value: Development
            - name: ASPNETCORE_HTTP_PORTS
              value: '80'
          image: <MY_ACCOUNT_ID>.dkr.ecr.<MY_REGION>.amazonaws.com/myrepository:1.0
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
        - name: otel-container
          image: amazon/aws-otel-collector:latest
          env:
            - name: AWS_REGION
              value: <MY_REGION>
          imagePullPolicy: Always
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
  name: mywebapi-service
  labels:
    app: mywebapi
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: http
      protocol: TCP
      name: http
  selector:
    app: mywebapi
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: mywebapi-scaledobject
spec:
  scaleTargetRef:
    name: mywebapi-deployment
    kind: Deployment
  minReplicaCount: 1
  maxReplicaCount: 10
  triggers:
  - type: aws-cloudwatch
    metadata:
      namespace: MyWebApi
      expression: SELECT AVG("http.server.active_requests") FROM MyWebApi
      targetMetricValue: "2"
      minMetricValue: "0"
      awsRegion: "<MY_REGION>"
      identityOwner: operator
```

* `ServiceAccount`: To interact with AWS CloudWatch, our pods needs to assume an AWS IAM Role through a service account.
    
* `Deployment`: Under this approach, the pods contains two containers: our application and the OTEL Collector.
    
* `Service`: Our application will be accessible through a service using a `LoadBalancer` as its type.
    
* `ScaledObject`: The [AWS CloudWatch](https://keda.sh/docs/2.12/scalers/aws-cloudwatch/) scaled object references the deployment created earlier using the `http.server.active_requests` mentioned previously.
    

Run the following command to deploy the application to the cluster:

```powershell
kubectl apply -f sidecar.yaml --namespace=<MY_K8S_NAMESPASE>
```

### Gateway

Create a `gateway.yaml` file with the following content:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
 name: mywebapi-sa
 annotations:
   eks.amazonaws.com/role-arn: arn:aws:iam::<MY_ACCOUNT_ID>:role/myrole
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: otel-deployment
  labels:
    app: otel
spec:
  replicas: 1
  selector:
    matchLabels:
      app: otel
  template:
    metadata:
      labels:
        app: otel
    spec:
      serviceAccountName: mywebapi-sa
      containers:
        - name: container
          image: amazon/aws-otel-collector:latest
          ports:
            - name: http
              containerPort: 4318
              protocol: TCP
            - name: grpc
              containerPort: 4317
              protocol: TCP
          env:
            - name: AWS_REGION
              value: <MY_REGION>
          imagePullPolicy: Always
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
  name: otel-service
  labels:
    app: otel
spec:
  type: ClusterIP
  ports:
    - port: 4318
      targetPort: 4318
      protocol: TCP
      name: http
    - port: 4317
      targetPort: 4317
      protocol: TCP
      name: grpc
  selector:
    app: otel
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mywebapi-deployment
  labels:
    app: mywebapi
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mywebapi
  template:
    metadata:
      labels:
        app: mywebapi
    spec:
      serviceAccountName: mywebapi-sa
      containers:
        - name: api-container
          env:
            - name: ASPNETCORE_ENVIRONMENT
              value: Development
            - name: ASPNETCORE_HTTP_PORTS
              value: '80'
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: http://otel-service:4318
            - name: OTEL_EXPORTER_OTLP_PROTOCOL
              value: http/protobuf
          image: <MY_ACCOUNT_ID>.dkr.ecr.<MY_REGION>.amazonaws.com/myrepository:1.0
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  name: mywebapi-service
  labels:
    app: mywebapi
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: http
      protocol: TCP
      name: http
  selector:
    app: mywebapi
---
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: mywebapi-scaledobject
spec:
  scaleTargetRef:
    name: mywebapi-deployment
    kind: Deployment
  minReplicaCount: 1
  maxReplicaCount: 10
  triggers:
  - type: aws-cloudwatch
    metadata:
      namespace: MyWebApi
      expression: SELECT AVG("http.server.active_requests") FROM MyWebApi
      targetMetricValue: "2"
      minMetricValue: "0"
      awsRegion: "us-east-2"
      identityOwner: operator
```

The main difference here is the presence of a dedicated deployment and service for the OTEL Collector, exposing the ports `4318` and `4317`. The deployment for our application includes two environment variables that specify the location of the OTEL Collector and the default protocol to use. Run the following command to deploy the application to the cluster:

```powershell
kubectl apply -f gateway.yaml --namespace=<MY_K8S_NAMESPASE>
```

## Tests

Create a `load.js` file with the following content:

```javascript
import http from 'k6/http';
import { sleep } from 'k6';
export const options = {
    vus: 50,
    duration: '600s',
};
export default function () {
    http.get('<MY_URL>/weatherforecast');
    sleep(1);
}
```

The URL of our service can be obtained with the following command:

```powershell
kubectl get services --namespace=<MY_K8S_NAMESPASE>
```

Execute the script using the command:

```powershell
k6 run load.js
```

After a couple of minutes, check the number of pods for our deployments using the command `kubectl get pods --namespace=<MY_K8S_NAMESPACE>` to see an output like:

```powershell
NAME                                      READY   STATUS    RESTARTS   AGE
mywebapi-deployment-754fc44b4b-6spfj      2/2     Running   0          23s
mywebapi-deployment-754fc44b4b-8bq7v      2/2     Running   0          39s
mywebapi-deployment-754fc44b4b-95vkg      2/2     Running   0          8m34s
mywebapi-deployment-754fc44b4b-nnwrx      2/2     Running   0          23s
mywebapi-deployment-754fc44b4b-qrvq9      2/2     Running   0          39s
mywebapi-deployment-754fc44b4b-xgds9      2/2     Running   0          39s
```

You can find the code and scripts [here](https://github.com/raulnq/keda-cloudwatch-otel). Thank you, and happy coding.