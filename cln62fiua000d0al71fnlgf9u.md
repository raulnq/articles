---
title: "Horizontal Pod Autoscaling with Kubernetes Event-Driven Autoscaler (KEDA)"
datePublished: Sat Sep 30 2023 13:25:20 GMT+0000 (Coordinated Universal Time)
cuid: cln62fiua000d0al71fnlgf9u
slug: horizontal-pod-autoscaling-with-kubernetes-event-driven-autoscaler-keda
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1695912760727/b7824474-f806-4a65-b6a5-0868940951b1.png
tags: kubernetes, net, autoscaling, keda

---

Autoscaling is a technique that automatically scales Kubernetes (K8s) workloads up or down, depending on historical resource usage. There are two pod-level auto-scalers:

* **Horizontal Pod Autoscaler (HPA)** adjusts the number of replicas of an application, typically based on CPU and memory usage.
    
* **Vertical Pod Autoscaler (VPA)** adjusts resource (CPU and memory) requests and limits of a container.
    

And one cluster-level auto-scaler:

* **Cluster Autoscaler** adjusts the number of cluster nodes based on all pod's requested resources.
    

Focusing on HPA, there are some drawbacks:

* **Limited scaling metrics**: By default, HPA primarily relies on CPU and memory usage and cannot respond to external events out-of-the-box, such as message queues or HTTP requests.
    
* **Slow reaction time**: HPA responds to CPU and memory usage rather than the application's workload, which can lead to delayed scaling decisions.
    

To address these drawbacks, we have [Kubernetes Event-driven Autoscaling](https://keda.sh/docs/2.12/) (KEDA).

> **KEDA** is a Kubernetes-based Event Driven Autoscaler. With KEDA, you can drive the scaling of any container in Kubernetes based on the number of events needing to be processed.

While [HPA](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/) is designed to scale based on resource utilization, KEDA is designed to scale based on event-driven workloads. KEDA is a lightweight component that can be executed in any Kubernetes cluster and works alongside the HPA.

## Pre-requisites

* Install [**Docker Desktop**](https://docs.docker.com/desktop/install/windows-install/)
    
* **Enable** [**Kubernetes**](https://docs.docker.com/desktop/kubernetes/) (the standalone version included in Docker Desktop)
    
* Ensure you have an [**Azure Account**](https://azure.microsoft.com/en-us/free/)
    
* Install [**Azure CLI**](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli)
    

## **Installation**

One of the easiest ways to [install KEDA](https://keda.sh/docs/2.12/deploy/) is by using Helm. Execute the following commands:

```powershell
helm repo add kedacore https://kedacore.github.io/charts
helm repo update
helm install keda kedacore/keda --namespace keda --create-namespace
```

Run `kubectl get deployments -n keda` to verify the installation:

```powershell
NAME                              READY   UP-TO-DATE   AVAILABLE   AGE
keda-admission-webhooks           1/1     1            1           2d22h
keda-operator                     1/1     1            1           2d22h
keda-operator-metrics-apiserver   1/1     1            1           2d22h
```

## Concepts

### Agent

The `keda-operator` activates and deactivates Kubernetes deployments to scale to and from zero on no events.

### Metrics Adapter

The `keda-operator-metrics-apiserver` is responsible for exposing external events to Kubernetes HPA.

### Event Sources

These are the external event sources by which KEDA scales the number of pods.

### Scalers

Event sources are monitored using [scalers](https://keda.sh/docs/2.12/scalers/).

### Admission Webhooks

The `keda-admission-webhooks` validate the resource changes to prevent misconfiguration. The Custom Resource Definitions (CRDs) provided by KEDA include:

* [ScaledObject](https://keda.sh/docs/2.12/concepts/scaling-deployments/): Specifies how KEDA should scale our application (either a deployment or statefulSet) and which triggers to utilize.
    
* [ScaledJob](https://keda.sh/docs/2.12/concepts/scaling-jobs/): This shares the same concept as a ScaledObject but is used to determine how many jobs are needed to process our workload.
    
* [TriggerAuthentication and ClusterTriggerAuthentication](https://keda.sh/docs/2.12/concepts/authentication/): These allow us to outline authentication parameters independently from the ScaledObject.
    

## Example

We will build two .NET applications: one for sending messages to an [Azure Service Bus](https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-messaging-overview) queue and another for consuming those messages.

### Creating the Service Bus Queue

Run the following commands to create a queue:

```powershell
az group create --name MyResourceGroup --location eastus
az servicebus namespace create --resource-group MyResourceGroup --name MyServiceBusNameSpace951 --location eastus
az servicebus queue create --resource-group MyResourceGroup --namespace-name MyServiceBusNameSpace951 --name MyQueue
```

The Azure Service Bus namespace must be unique. Use the following command to get the connection string:

```powershell
az servicebus namespace authorization-rule keys list --resource-group MyResourceGroup --namespace-name MyServiceBusNameSpace951 --name RootManageSharedAccessKey --query primaryConnectionString --output tsv
```

### Building the Application

Run the following commands to set up the solution:

```powershell
dotnet new console -n MyProducer
dotnet new worker -n MyConsumer
dotnet new sln -n Keda
dotnet sln add --in-root MyProducer
dotnet sln add --in-root MyConsumer
dotnet add MyProducer package Azure.Messaging.ServiceBus
dotnet add MyConsumer package Azure.Messaging.ServiceBus
```

Open the solution, navigate to the `MyProducer` project, and update the `Program.cs` file as follows:

```csharp
using Azure.Messaging.ServiceBus;

await using (var client = new ServiceBusClient("<MY_QUEUE_CONNECTION_STRING>"))
{
    await using (var sender = client.CreateSender("MyQueue"))
    {
        var numOfMessages = 1000;
        using (ServiceBusMessageBatch messageBatch = await sender.CreateMessageBatchAsync())
        {
            for (int i = 1; i <= numOfMessages; i++)
            {
                if (!messageBatch.TryAddMessage(new ServiceBusMessage($"Message {i}")))
                {
                    throw new Exception($"The message {i} is too large to fit in the batch.");
                }
            }
            await sender.SendMessagesAsync(messageBatch);
            Console.WriteLine($"A batch of {numOfMessages} messages has been published to the queue.");
        }
    }
}
```

At the project level, create a `Dockefile` file with the following content:

```bash
FROM mcr.microsoft.com/dotnet/aspnet:6.0 AS base
WORKDIR /app

FROM mcr.microsoft.com/dotnet/sdk:6.0 AS build
COPY ["MyProducer/MyProducer.csproj", "MyProducer/"]
RUN dotnet restore "MyProducer/MyProducer.csproj"
COPY . .
WORKDIR "/MyProducer"
RUN dotnet build "MyProducer.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "MyProducer.csproj" -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "MyProducer.dll"]
```

The producer application will run as a job in the Kubernetes cluster, so create a `job.yml` file as follows:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: myproducer-job
spec:
  template:
    spec:
      containers:
      - name: myproducer-container
        image:  raulnq/myproducer:1.0
        imagePullPolicy: IfNotPresent
      restartPolicy: Never
  backoffLimit: 4
```

Navigate to the `MyConsumser` project and update the `Worker.cs` file as follows:

```csharp
using Azure.Core;
using Azure.Messaging.ServiceBus;

namespace MyConsumer;

public class Worker : IHostedService
{
    ServiceBusClient client;

    ServiceBusProcessor processor;

    public Worker()
    {
        client = new ServiceBusClient("<MY_QUEUE_CONNECTION_STRING>");
        processor = client.CreateProcessor("MyQueue", new ServiceBusProcessorOptions());
    }

    public async Task StartAsync(CancellationToken cancellationToken)
    {
        processor.ProcessMessageAsync += MessageHandler;
        processor.ProcessErrorAsync += ErrorHandler;
        await processor.StartProcessingAsync();
    }

    public async Task StopAsync(CancellationToken cancellationToken)
    {
        await processor.StopProcessingAsync();
        await processor.CloseAsync();
        await processor.DisposeAsync();
        await client.DisposeAsync();
    }

    async Task MessageHandler(ProcessMessageEventArgs args)
    {
        string body = args.Message.Body.ToString();
        Console.WriteLine($"Received: {body}");
        await args.CompleteMessageAsync(args.Message);
    }

    Task ErrorHandler(ProcessErrorEventArgs args)
    {
        Console.WriteLine(args.Exception.ToString());
        return Task.CompletedTask;
    }
}
```

At the project level, create a `Dockefile` file with the following content:

```bash
FROM mcr.microsoft.com/dotnet/aspnet:6.0 AS base
WORKDIR /app

FROM mcr.microsoft.com/dotnet/sdk:6.0 AS build
COPY ["MyConsumer/MyConsumer.csproj", "MyConsumer/"]
RUN dotnet restore "MyConsumer/MyConsumer.csproj"
COPY . .
WORKDIR "/MyConsumer"
RUN dotnet build "MyConsumer.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "MyConsumer.csproj" -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "MyConsumer.dll"]
```

The consumer application will run as a deployment in the Kubernetes cluster, so create a `deployment.yml` file as follows:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myconsumer-deployment
  labels:
    app: myconsumer
spec:
  replicas: 1
  selector:
    matchLabels:
      app: myconsumer
  template:
    metadata:
      labels:
        app: myconsumer
    spec:
      containers:
        - name: myconsumer-container
          imagePullPolicy: IfNotPresent
          image: raulnq/myconsumer:1.0
```

### Deploying the Application to the Cluster

Before deploying, we need to create the images for our application by executing the following commands at the solution level:

```powershell
docker build -t raulnq/myconsumer:1.0 -f .\MyConsumer\Dockerfile .
docker build -t raulnq/myproducer:1.0 -f .\MyProducer\Dockerfile .
```

Now, we are ready to deploy our consumer application using the following command:

```powershell
kubectl apply -f .\MyConsumer\deployment.yml
```

The producer application can be deployed using the following command:

```powershell
kubectl apply -f .\MyProducer\job.yml
```

However, we will wait until the final step to deploy it.

### AutoScaling Setup

We will follow the [official documentation](https://keda.sh/docs/2.12/scalers/azure-service-bus/) to set up the scaler. Navigate to the `MyConsumser` project and create a `secret.yml` file as follows:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: myconsumer-secret
data:
  connectionstring: <MY_QUEUE_CONNECTION_STRING_BASE64>
```

We can use this command to get the connection string in Base64 format:

```powershell
$base64 = [Convert]::ToBase64String([Text.Encoding]::UTF8.GetBytes('<MY_QUEUE_CONNECTION_STRING>'))
Write-Output $base64
```

Create a `triggerauthentication.yml` file as follows:

```yaml
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: myconsumer-trigger-authentication
spec:
  secretTargetRef:
  - parameter: connection
    name: myconsumer-secret
    key: connectionstring
```

Create a `scaledobject.yml` file containing the following content:

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: myconsumer-scaler
spec:
  scaleTargetRef:
    name: myconsumer-deployment
  minReplicaCount: 1
  maxReplicaCount: 10
  triggers:
  - type: azure-servicebus
    metadata:
      queueName: MyQueue
      queueLength: '10'
    authenticationRef:
      name: myconsumer-trigger-authentication
```

Run `kubectl get scaledobjects` to verify our setup:

```powershell
NAME                SCALETARGETKIND      SCALETARGETNAME         MIN   MAX   TRIGGERS           AUTHENTICATION                      READY   ACTIVE   FALLBACK   PAUSED    AGE
myconsumer-scaler   apps/v1.Deployment   myconsumer-deployment   1     10    azure-servicebus   myconsumer-trigger-authentication   True    False    False      Unknown   12h
```

### Testing the AutoScaling

Execute the command `kubectl apply -f ./MyProducer/job.yml`, wait for a few seconds, and then verify the number of pods for our application:

```powershell
NAME                                                         READY   STATUS      RESTARTS      AGE
myconsumer-deployment-85d55cd6d8-44tsh                       1/1     Running     0             57s
myconsumer-deployment-85d55cd6d8-7c27b                       1/1     Running     0             45m
myconsumer-deployment-85d55cd6d8-dllh9                       1/1     Running     0             117s
myconsumer-deployment-85d55cd6d8-h5bb6                       1/1     Running     0             117s
myconsumer-deployment-85d55cd6d8-lgxcv                       1/1     Running     0             117s
myconsumer-deployment-85d55cd6d8-mmgn2                       1/1     Running     0             57s
myconsumer-deployment-85d55cd6d8-pvz69                       1/1     Running     0             57s
myconsumer-deployment-85d55cd6d8-wf8gm                       1/1     Running     0             57s
myproducer-job-t7hg2                                         0/1     Completed   0             2m59s
```

That's it; our event-based autoscaling is now up and running.

## Conclusion

Kubernetes Event-Driven Autoscaler (KEDA) addresses the limitations of the Horizontal Pod Autoscaler by providing event-driven autoscaling for containerized applications. By integrating with various event sources and working alongside HPA, KEDA allows efficient and timely scaling of workloads within Kubernetes clusters. The example provided demonstrates how to set up and test autoscaling with Azure Service Bus, showcasing KEDA's potential in real-world scenarios. All the code is available [here](https://github.com/raulnq/k8s-keda). Thanks, and happy coding.