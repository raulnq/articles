---
title: "Scaling Amazon Elastic Kubernetes Service Workloads with KEDA and Amazon CloudWatch"
datePublished: Sun Jan 07 2024 16:07:19 GMT+0000 (Coordinated Universal Time)
cuid: clr3ot690000108kyhzuv9gpa
slug: scaling-amazon-elastic-kubernetes-service-workloads-with-keda-and-amazon-cloudwatch
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1703120504815/cec0e009-cf81-4639-907b-e54c62f0df3d.png
tags: aws, eks, aws-cloudwatch, keda, eks-cluster

---

In our previous post, [Horizontal Pod Autoscaling with Kubernetes Event-Driven Autoscaler](https://blog.raulnq.com/horizontal-pod-autoscaling-with-kubernetes-event-driven-autoscaler-keda), we explored how to scale an application based on events (Azure Service Bus). Today, we aim to scale an application running in Amazon EKS based on a metric (requests per second) provided by Amazon CloudWatch.

## Pre-requisites

* Install [Docker Desktop](https://docs.docker.com/desktop/install/windows-install/)
    
* Enable [Kubernetes](https://docs.docker.com/desktop/kubernetes/) (the standalone version included in Docker Desktop)
    
* An [Amazon EKS Cluster](https://docs.aws.amazon.com/eks/latest/userguide/clusters.html) with [KEDA](https://keda.sh/docs/2.12/) installed is required. We will use the option `operator` as the `identityOwner` for our [AWS CloudWatch](https://keda.sh/docs/2.12/scalers/aws-cloudwatch/) scaler. Therefore, we must grant the KEDA operator the necessary IAM permissions to access CloudWatch. You can find an example of how to accomplish this [here](https://dev.to/carlocolumna/keda-in-amazon-eks-part-2-scale-based-on-aws-sqs-queue-5g4d).
    
* Docker Desktop Kubernetes context configured to work with the Amazon EKS cluster.
    
* An [IAM User](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html#id_users_create_console) with programmatic access.
    
* Install the [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-creds).
    
* Install [Terraform CLI](https://learn.hashicorp.com/tutorials/terraform/install-cli).
    
* Install [K6](https://k6.io/docs/get-started/installation/).
    

## The Application

Run the following commands to set up the solution:

```powershell
dotnet new web -o MyApi
dotnet new sln -n MyApi
dotnet sln add --in-root MyApi
dotnet add MyApi package AWSSDK.CloudWatch
dotnet add MyApi package AWSSDK.Extensions.NETCore.Setup
dotnet add MyApi package AWSSDK.SecurityToken
```

We will create an [EventListener](https://learn.microsoft.com/en-us/dotnet/api/system.diagnostics.tracing.eventlistener?view=net-8.0) class to subscribe to the `Microsoft.AspNetCore.Hosting` [source](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/available-counters). Then, the `requests-per-second` metric will be forwarded to a [Channel](https://learn.microsoft.com/en-us/dotnet/core/extensions/channels) responsible for sending it to AWS CloudWatch. Open the solution and create the `RequestPerSecondChannelProducer.cs` file with the following content:

```csharp
using System.Diagnostics.Tracing;
using System.Threading.Channels;

public class RequestPerSecondChannelProducer : EventListener
{
    private readonly Channel<string> _channel;

    public RequestPerSecondChannelProducer(Channel<string> channel)
    {
        _channel = channel;
    }

    protected override void OnEventSourceCreated(EventSource source)
    {
        if (source.Name.Equals("Microsoft.AspNetCore.Hosting"))
        {
            EnableEvents(source, EventLevel.Verbose, EventKeywords.All, new Dictionary<string, string?>()
            {
                ["EventCounterIntervalSec"] = "1"
            });
        }
    }

    protected override void OnEventWritten(EventWrittenEventArgs eventData)
    {
        if (eventData?.EventName!=null && eventData.EventName.Equals("EventCounters", StringComparison.InvariantCulture) && eventData.Payload!=null)
        {
            for (int i = 0; i < eventData.Payload.Count; i++)
            {
                var eventPayload = eventData.Payload[i] as IDictionary<string, object>;

                if (eventPayload != null && eventPayload.Any(item=> item.Value!=null && item.Value.ToString()!.Equals("requests-per-second", StringComparison.InvariantCulture)))
                {
                    var value = eventPayload.First(item => item.Key != null && item.Key.Equals("Increment", StringComparison.InvariantCulture));

                    if (value.Value != null)
                    {
                        _channel.Writer.TryWrite(value.Value.ToString()!);
                    }
                }
            }
        }
    }
}
```

Create a `CloudWatchChannelConsumer.cs` file as follows:

```csharp
using Amazon.CloudWatch;
using Amazon.CloudWatch.Model;
using System.Threading.Channels;

public class CloudWatchChannelConsumer : BackgroundService
{
    private readonly Channel<string> _channel;

    private readonly IAmazonCloudWatch _amazonCloudWatch;

    private readonly string _host;

    private readonly ILogger<CloudWatchChannelConsumer> _logger;

    public CloudWatchChannelConsumer(Channel<string> channel, IAmazonCloudWatch amazonCloudWatch, ILogger<CloudWatchChannelConsumer> logger)
    {
        _channel = channel;
        _host = Environment.GetEnvironmentVariable("POD_NAME") ?? "local";
        _logger = logger;
        _amazonCloudWatch = amazonCloudWatch;
    }
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!_channel.Reader.Completion.IsCompleted && await _channel.Reader.WaitToReadAsync())
        {
            if (_channel.Reader.TryRead(out var value))
            {
                try
                {
                    await _amazonCloudWatch.PutMetricDataAsync(new PutMetricDataRequest
                    {
                        Namespace = "MyApiNamespace",
                        MetricData =
                    [
                        new MetricDatum
                        {
                            MetricName = "RequestPerSecond",
                            Value = double.Parse(value),
                            Unit = StandardUnit.CountSecond,
                            TimestampUtc = DateTime.UtcNow,
                            Dimensions = new List<Dimension>
                            {
                                new() {
                                    Name = "Host",
                                    Value = _host
                                }
                            }
                        }
                    ]
                    });
                }
                catch (Exception ex)
                {
                    _logger.LogError(ex, "Failed to send CloudWatch Metric");
                }
            }
        }
    }
}
```

We do not recommend using the [AWS SDK for .NET](https://docs.aws.amazon.com/sdk-for-net/v3/developer-guide/csharp_cloudwatch_code_examples.html) to send metrics to CloudWatch. Instead, the proper approach would be to use the OTEL Collector, as shown [here](https://blog.raulnq.com/amazon-eks-deploy-otel-collector-as-sidecar). Open the `Program.cs` file and update the content as follows:

```csharp
using Amazon.CloudWatch;
using System.Threading.Channels;
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddDefaultAWSOptions(builder.Configuration.GetAWSOptions());
builder.Services.AddAWSService<IAmazonCloudWatch>();
builder.Services.AddHostedService<CloudWatchChannelConsumer>();
var channel = Channel.CreateUnbounded<string>();
builder.Services.AddSingleton(channel);
var app = builder.Build();
app.MapGet("/", () => "Hello World!");
var client = app.Services.GetRequiredService<IAmazonCloudWatch>();
var listener = new RequestPerSecondChannelProducer(channel);
app.Run();
```

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

data "aws_iam_policy" "cloudwatch_policy" {
  name = "CloudWatchFullAccess"
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
    data.aws_iam_policy.cloudwatch_policy.arn,
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

We are creating an [Amazon ECR](https://docs.aws.amazon.com/AmazonECR/latest/userguide/Repositories.html) repository to upload our application's image and an [IAM Role](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html) for our Pod with sufficient permissions for AWS CloudWatch. Run the following commands to create the resources in AWS:

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

To interact with the AWS CloudWatch API, we must assume an AWS IAM Role from our Pod through a Service Account. You can find more information [here](https://blog.raulnq.com/how-to-assume-an-aws-iam-role-from-an-eks-pod). Create a `service.yaml` file with the following content:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
 name: myapi-sa
 annotations:
   eks.amazonaws.com/role-arn: arn:aws:iam::<MY_ACCOUNT_ID>:role/myrole
```

Next, we will use the Service Account mentioned earlier. Create a `deployment.yaml` file with the following content:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapi-deployment
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
      serviceAccountName: myapi-sa
      containers:
        - name: myapi-c
          env:
            - name: ASPNETCORE_ENVIRONMENT
              value: Development
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
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
  name: myapi-service
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

Finally, create a [Scaled Object](https://keda.sh/docs/2.12/scalers/aws-cloudwatch/) referencing the Deployment created earlier. Create a `scaledobject.yaml` file containing the following content:

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: myapi-scaledobject
spec:
  scaleTargetRef:
    name: myapi-deployment
    kind: Deployment
  minReplicaCount: 1
  maxReplicaCount: 10
  triggers:
  - type: aws-cloudwatch
    metadata:
      namespace: MyApiNamespace
      expression: SELECT AVG(RequestPerSecond) FROM MyApiNamespace
      targetMetricValue: '2'
      awsRegion: "<MY_REGION>"
      identityOwner: operator
```

We use the `expression` property to specify a query that should remain below the one indicated by the `targetMetricValue` property. In our case, the query provides the average `request-per-second` for all the Pods in our Deployment. Execute the following commands to deploy the application to the cluster:

```powershell
kubectl apply -f serviceaccount.yaml --namespace=<MY_K8S_NAMESPASE>
kubectl apply -f deployment.yaml --namespace=<MY_K8S_NAMESPASE>
kubectl apply -f service.yaml --namespace=<MY_K8S_NAMESPASE>
kubectl apply -f scaledobject.yaml --namespace=<MY_K8S_NAMESPASE>
```

Run `kubectl get services --namespace=<MY_K8S_NAMESPACE>` to see the URL assigned to our application. The output will look something like this:

```powershell
NAME           TYPE           CLUSTER-IP       EXTERNAL-IP    PORT(S)        AGE
myapi-service  LoadBalancer   10.100.136.188   <MY_URL>       80:31119/TCP   18d
```

## Tests

Create a `load.js` file with the following content:

```javascript
import http from 'k6/http';
import { sleep } from 'k6';
export const options = {
    vus: 16,
    duration: '600s',
};
export default function () {
    http.get('<MY_URL>');
    sleep(1);
}
```

Execute the script using the command `k6 run load.js`. Increasing the load on the application will cause the `request-per-second` metric in AWS CloudWatch to rise to two or higher, triggering the deployment scaling.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1704640263284/d8ea4926-9869-4ffc-91fe-86f24a58edfc.png align="center")

Next, we can observe how the Deployment scales up to create more pods in response to the load on the application. Run `kubectl get pods --namespace=<MY_K8S_NAMESPACE>`:

```powershell
NAME                                  READY   STATUS    RESTARTS         AGE
myapi-deployment-7bbdb8f8d9-44vds     1/1     Running   0                6m46s
myapi-deployment-7bbdb8f8d9-48567     1/1     Running   0                39h
myapi-deployment-7bbdb8f8d9-frpcf     1/1     Running   0                6m46s
```

In conclusion, scaling workloads in Amazon EKS can be effectively achieved using KEDA and Amazon CloudWatch. This approach enables an application to scale based on specific metrics, ensuring optimal resource utilization and improved application performance. You can find the code and scripts [here](https://github.com/raulnq/keda-cloudwatch). Thank you, and happy coding.