---
title: "Scaling  Kubernetes Pods Based on  HTTP Traffic using KEDA HTTP Add-on"
datePublished: Sun Jun 23 2024 06:44:09 GMT+0000 (Coordinated Universal Time)
cuid: clxr6o1y4000009jkf6spbwfa
slug: scaling-kubernetes-pods-based-on-http-traffic-using-keda-http-add-on
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1719005221949/9f7f1783-e5d2-4235-b097-b930be685138.png
tags: scaling, kubernetes, net, keda

---

In a previous [article](https://blog.raulnq.com/scaling-amazon-elastic-kubernetes-service-workloads-with-keda-otel-collector-and-amazon-cloudwatch), we demonstrated how to scale pods based on HTTP traffic using [KEDA](https://keda.sh/docs/2.14/) and [CloudWatch](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/WhatIsCloudWatch.html). We utilized CloudWatch to store and retrieve the metrics for our application and, based on that, scale the number of pods. But is there a way to achieve the same result using only KEDA? The answer is yes, with the [KEDA HTTP add-on](https://github.com/kedacore/http-add-on).

> The KEDA HTTP Add-on allows Kubernetes users to automatically scale their HTTP servers up and down (including to/from zero) based on incoming HTTP traffic.

## How does it work?

A typical application on Kubernetes uses an ingress to route requests to a service, which then routes them to the pods for handling.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1719012860971/1f3eef59-70e6-443b-b0c3-bdb2f9a8848e.png align="center")

There are three [main components](https://github.com/kedacore/http-add-on/blob/main/docs/design.md) provided by the KEDA HTTP Add-on:

* **Operator**: This component listens for events related to `HTTPScaledObject` resources and creates, updates, or removes internal mechanisms as needed.
    
* **Interceptor**: This component accepts and routes external HTTP traffic to the appropriate application.
    
* **Scaler**: This component monitors the size of the pending HTTP request queue for a given application and reports it to KEDA.
    

As a result, once the KEDA HTTP Add-on is installed in our cluster, we need to route the traffic from the ingress to the interceptor manually. The interceptor will then route the request back to our original service.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1719061744124/0e70f30a-191d-4cf9-b941-e48c10f36f44.png align="center")

## Pre-requisites

* Install [Docker Desktop](https://docs.docker.com/desktop/install/windows-install/)
    
* [Enable Kubernetes](https://docs.docker.com/desktop/install/windows-install/) (the standalone version included in Docker Desktop)
    
* Install [Helm CLI](https://helm.sh/docs/intro/install/)
    
* Follow the instructions in the post [Testing the NGINX Ingress Controller locally](https://blog.raulnq.com/testing-the-nginx-ingress-controller-locally-docker-desktop) to set up the ingress.
    
* Enable the Kubernetes [Metrics Server](https://github.com/kubernetes-sigs/metrics-server) on Docker Desktop. Run the command `kubectl apply -f components.yaml`. You can find the `components.yaml` file [here](https://github.com/raulnq/keda-http-add-on/blob/main/components.yaml).
    
* Follow the instructions in the post [Horizontal Pod Autoscaling with Kubernetes Event-Driven Autoscaler](https://blog.raulnq.com/horizontal-pod-autoscaling-with-kubernetes-event-driven-autoscaler-keda) to install KEDA.
    
* Install [K6](https://k6.io/docs/get-started/installation/).
    

## Installation

The [KEDA HTTP Add-on](https://github.com/kedacore/charts/tree/main/http-add-on) uses Helm for installation in the cluster. Execute the following command:

```powershell
helm install http-add-on kedacore/keda-add-ons-http --namespace keda --set interceptor.responseHeaderTimeout=5s
```

We are customizing the default installation by passing the parameter `--set interceptor.responseHeaderTimeout=5s` on the command line. Since we are introducing a random delay in our application between 0 and 1.5 seconds, this setup will prevent a `Bad Gateway` response from the interceptor to the caller. The complete list of parameters and descriptions can be found [here](https://github.com/kedacore/charts/tree/main/http-add-on#configuration).

> The above command installed the HTTP Add-on in cluster-global mode. Add `--set operator.watchNamespace=<target namespace>` to install the HTTP Add-on in namepaced mode. If you do this, you must also install KEDA in namespaced mode and use the same target namespace.

## The Application

Run the following commands to set up the solution:

```powershell
dotnet new webapi -n MyWebApi
dotnet new sln -n MyWebApi
dotnet sln add --in-root MyWebApi
```

Open the `Program.cs` file and update the content as follows:

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();
var app = builder.Build();
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

A random delay is added to each request to simulate slower processing time.

## Docker Image

At project level, create a `Dockerfile` with the following content:

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
USER app
WORKDIR /app
EXPOSE 8080
EXPOSE 8081

FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
ARG BUILD_CONFIGURATION=Release
WORKDIR /src
COPY ["MyWebApi/MyWebApi.csproj", "MyWebApi/"]
RUN dotnet restore "./MyWebApi/MyWebApi.csproj"
COPY . .
WORKDIR "/src/MyWebApi"
RUN dotnet build "./MyWebApi.csproj" -c $BUILD_CONFIGURATION -o /app/build

FROM build AS publish
ARG BUILD_CONFIGURATION=Release
RUN dotnet publish "./MyWebApi.csproj" -c $BUILD_CONFIGURATION -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "MyWebApi.dll"]
```

Build the image by running the following command at the solution level:

```powershell
docker build -t raulnq/mywebapi:1.0 -f .\MyWebApi\Dockerfile .
```

## Kubernetes

Create a `resources.yaml` file with the following content:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-deployment
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
      containers:
        - name: api-container
          image: raulnq/mywebapi:1.0
          imagePullPolicy: IfNotPresent
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
  name: api-service
  labels:
    app: api
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: 8080
      protocol: TCP
      name: http
  selector:
    app: api
---
```

The first part of the file contains standard `Deployment` and `Service` definitions. Let's add the following content:

```yaml
kind: Service
apiVersion: v1
metadata:
  name: keda-interceptor-proxy
spec:
  type: ExternalName
  externalName: keda-add-ons-http-interceptor-proxy.keda.svc.cluster.local
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: www.api.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: keda-interceptor-proxy
            port:
              number: 8080
---
```

In this section, we want to set up the ingress rule to route requests to the `keda-add-ons-http-interceptor-proxy` service. However, there is a problem: the service is in a different namespace, and the Kubernetes [documentation](https://kubernetes.io/docs/concepts/services-networking/ingress/) states:

> A `Resource` backend is an ObjectRef to another Kubernetes resource within the same namespace as the Ingress object.

The solution is to create a service with the type [`ExternalName`](https://kubernetes.io/docs/concepts/services-networking/service/#externalname) in our namespace, referencing the `keda-add-ons-http-interceptor-proxy` service. Then, we can use this service in the ingress rule. The final part is the `HTTPScaledObject` resource:

```yaml
kind: HTTPScaledObject
apiVersion: http.keda.sh/v1alpha1
metadata:
    name: api-scaledobject
spec:
    hosts:
        - www.api.com
    scaleTargetRef:
        name: api-deployment
        kind: Deployment
        apiVersion: apps/v1
        service: api-service
        port: 80
    replicas:
        min: 1
        max: 10
    scalingMetric:
        requestRate:
            granularity: 1s
            targetValue: 5
            window: 1m
```

The `HTTPScaledObject` is a custom resource definition (CRD) that allows us to configure HTTP-based autoscaling for our applications:

* `host`: These are the hosts to which this scaling rule will apply.
    
* `scaleTargetRef`: Describes which workload to scale and the service to route HTTP traffic to.
    
    * `name` and `kind`: Defines the workload to scale, in this case, a `Deployment`.
        
    * `service` and `port`: This is the service to route traffic to.
        
    * `apiVersion`: This is the API version of the workload to scale.
        
* `scalingMetric`: Describes how the workload should scale.
    
    * `requestRate`
        
        * `targetValue`: Defines the value for the scaling configuration.
            
        * `windows`: Defines the aggregation window for the request rate calculation.
            
        * `granularity`: Defines the granularity of the aggregated requests for the request rate calculation.
            

The complete specification can be found [here](https://github.com/kedacore/http-add-on/blob/main/docs/ref/v0.8.0/http_scaled_object.md). Run the following command to deploy the resources to the cluster:

```powershell
kubectl apply -f resources.yaml
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
    http.get('http://www.api.com/weatherforecast');
    sleep(1);
}
```

Execute the script using the command:

```powershell
k6 run load.js
```

After a couple of minutes, check the number of pods for our deployments using the command `kubectl get pods` to see an output like:

```powershell
NAME                                          READY   STATUS    RESTARTS   AGE
api-deployment-8486cd99df-4cbp6               1/1     Running   0          14h
api-deployment-8486cd99df-69w4w               1/1     Running   0          40s
api-deployment-8486cd99df-d5zzp               1/1     Running   0          40s
api-deployment-8486cd99df-rhsxr               1/1     Running   0          40s
```

KEDA HTTP Add-on is still in beta and not yet recommended for production use. We need to value the benefits against the risks. It's important to note that with this approach, we are adding a new piece of software to our system: the proxy. Therefore, we must monitor and size it according to our workloads. You can find the code and scripts [here](https://github.com/raulnq/keda-http-add-on). Thank you, and happy coding.