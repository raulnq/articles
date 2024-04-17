---
title: "Amazon EKS: Deploy OTEL Collector as sidecar"
datePublished: Fri Sep 16 2022 14:53:32 GMT+0000 (Coordinated Universal Time)
cuid: cl84lp3ni05eor2nv5q7i226s
slug: amazon-eks-deploy-otel-collector-as-sidecar
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1663306198037/y_LXokEVZ.png
tags: aws, kubernetes, net, eks, opentelemetry

---

In a previous post, [OpenTelemetry: The OTEL Collector](https://blog.raulnq.com/opentelemetry-the-otel-collector), we saw how easy it's to use the OTEL Collector with a .NET application. Today we will see how to deploy those components in an Amazon EKS Cluster. To complete the post, we will require:

* An [IAM User](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html#id_users_create_console) with programmatic access.
    
* Setup the [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html) locally.
    
* An [Amazon EKS Cluster](https://docs.aws.amazon.com/eks/latest/userguide/clusters.html) available.
    

Download the code located [here](https://github.com/raulnq/opentelemetry-sandbox/tree/otel). We will need to adjust it to support [AWS X-Ray](https://docs.aws.amazon.com/xray/latest/devguide/aws-xray.html) as a backend ([here](https://github.com/open-telemetry/opentelemetry-dotnet-contrib/tree/main/src/OpenTelemetry.Contrib.Extensions.AWSXRay) you can find the reason why). Add the following NuGet Package to the project:

* [OpenTelemetry.Contrib.Extensions.AWSXRay](https://www.nuget.org/packages/OpenTelemetry.Contrib.Extensions.AWSXRay/)
    

Open the `Program.cs` file and change the following section:

```csharp
...
builder.Services.AddOpenTelemetryTracing(builder =>
{
    builder
    .AddXRayTraceId()
    .AddOtlpExporter()
    .AddSource("AnimeQuoteApi")
    .SetResourceBuilder(ResourceBuilder.CreateDefault().AddService("AnimeQuoteApi"))
    .AddHttpClientInstrumentation()
    .AddAspNetCoreInstrumentation()
    ;
});
...
```

And that's it from a code perspective. Now, let's work on generating the Docker image. Go to the `animequoteapi` folder and add a `Dockerfile` file with the following content:

```powershell
FROM mcr.microsoft.com/dotnet/aspnet:6.0 AS base
WORKDIR /app
EXPOSE 80

FROM mcr.microsoft.com/dotnet/sdk:6.0 AS build
WORKDIR /src
COPY ["AnimeQuoteApi.csproj", "."]
RUN dotnet restore "AnimeQuoteApi.csproj"
COPY . .
WORKDIR "/src"
RUN dotnet build "AnimeQuoteApi.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "AnimeQuoteApi.csproj" -c Release -o /app/publish /p:UseAppHost=false

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "AnimeQuoteApi.dll"]
```

To store the image, we will use [Amazon ECR](https://docs.aws.amazon.com/AmazonECR/latest/userguide/what-is-ecr.html) as a container image registry. Go to the AWS Console and create a new repository (Amazon ECR -&gt; Create Repository) named `opentelemetry-api` :

![repository.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1663284343044/wRdBXUNjj.png align="left")

To upload our image, run the following commands:

```powershell
docker build -t [account-id].dkr.ecr.[region].amazonaws.com/opentelemetry-api:1.0 .
aws ecr get-login-password --region [region] | docker login --username AWS --password-stdin [account-id].dkr.ecr.[region].amazonaws.com
docker push [account-id].dkr.ecr.[region].amazonaws.com/opentelemetry-api:1.0
```

* `[region]` is the AWS Region for the database cluster. An example of a region is us-east-2.
    
* `[account-id]` is the AWS account number. An example of an account number is 1234567890.
    

![readrepo.PNG](https://cdn.hashnode.com/res/hashnode/image/upload/v1663285347746/bC_f3GUu9.PNG align="left")

Go to the AWS Console and create a new [AIM Policy](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_create.html) (AIM-&gt; Policies-&gt; Create policy) named `opentelemetry-policy` with the following json:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
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
            ],
            "Resource": "*"
        }
    ]
}
```

![createpolicy.PNG](https://cdn.hashnode.com/res/hashnode/image/upload/v1663288366844/OTBSb3y1Q.PNG align="left")

Then, create an AIM Role named `opentelemetry-role` as we saw in the post [How to assume an AWS IAM Role from an EKS Pod?](https://blog.raulnq.com/how-to-assume-an-aws-iam-role-from-an-eks-pod). The final IAM Role should have a thrust policy like this:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "",
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws:iam::[account-id]:oidc-provider/oidc.eks.[region].amazonaws.com/id/[oidc-id]"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringLike": {
                    "oidc.eks.[region].amazonaws.com/id/[oidc-id]:sub": "system:serviceaccount:[k8s-namespace]:*"
                }
            }
        }
    ]
}
```

* `[oidc-id]` is part of the OpenID Connect provider URL in our EKS Cluster.
    
* `[k8s-namespace]` is the namespace in our EKS Cluster where we will deploy the application.
    

Create a `service-account.yaml` file with the following content:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
 name: opentelemetry-sa
 annotations:
   eks.amazonaws.com/role-arn:  arn:aws:iam::[account-id]:role/opentelemetry-role
```

The OTEL Collector [image](https://github.com/aws-observability/aws-otel-collector) that we will use is a distribution supported by AWS. Create a `deployment.yaml` file with the following content:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: opentelemetry-deployment
  labels:
    app: api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      serviceAccountName: opentelemetry-sa
      containers:
        - name: api-container
          env:
            - name: ASPNETCORE_ENVIRONMENT
              value: Development
          image: [account-id].dkr.ecr.[region].amazonaws.com/opentelemetry-api:2.0
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
        - name: otel-container
          image: amazon/aws-otel-collector:latest
          env:
            - name: AWS_REGION
              value: [region]
          imagePullPolicy: Always
          resources:
            limits:
              cpu: 500m
              memory: 500Mi
            requests:
              cpu: 250m
              memory: 250Mi
```

And finally, create a `service.yaml` file with the following content:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: opentelemetry-service
  labels:
    app: api
spec:
  type: LoadBalancer
  ports:
    - port: 80
      targetPort: http
      protocol: TCP
      name: http
  selector:
    app: api
```

Time to deploy everything to our EKS Cluster, run the following commands:

```powershell
kubectl apply -f service-account.yml --namespace=[k8s-namespace]
kubectl apply -f deployment.yml --namespace=[k8s-namespace]
kubectl apply -f service.yml --namespace=[k8s-namespace]
```

Once it is completed, we can check the URL of our application with:

```powershell
kubectl get services --namespace=[k8s-namespace]
```

Look for the `EXTERNAL-IP` column, open the swagger page (add `/swagger/index.html` add the end of the URL) in a browser and send a few requests to our API:

![swagger.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1663292404418/NhHPdqmqF.png align="left")

Go to the AWS Console to see the [Service map](https://docs.aws.amazon.com/xray/latest/devguide/xray-console.html) graph (CloudWatch-&gt;X-Ray traces-&gt;Service map):

![sevicemap.PNG](https://cdn.hashnode.com/res/hashnode/image/upload/v1663292654576/-HJaGED01.PNG align="left")

Go to Traces (CloudWatch-&gt;X-Ray traces-&gt;Traces):

![tracesb.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1663292785815/fwLgnqvw2.png align="left")

And open a record to see all the details:

![trace.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1663292907434/sG7lgRT-n.png align="left")

Go to Metrics (CloudWatch-&gt;Metrics-&gt;All metrics), and open the custom namespace `AnimeQuoteApi`:

![metrics.PNG](https://cdn.hashnode.com/res/hashnode/image/upload/v1663304117191/WCl-gdywQ.PNG align="left")

Select one metric:

![metric.PNG](https://cdn.hashnode.com/res/hashnode/image/upload/v1663304426349/QWzRiudHL.PNG align="left")

In case we need to use our own OTEL Collector configuration file, we could create a ConfigMap with the following content:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: opentelemetry-config
data:
  config.yaml: |
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
    processors:
      batch/traces:
        timeout: 1s
        send_batch_size: 50
      batch/metrics:
        timeout: 60s
      batch:
    exporters:
      awsxray:
      awsemf:
    service:
      pipelines:
        traces:
          receivers: [otlp]
          processors: [batch/traces]
          exporters: [awsxray]
        metrics:
          receivers: [otlp]
          processors: [batch/metrics]
          exporters: [awsemf]
```

And attach it to our pod:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: opentelemetry-deployment
  labels:
    app: api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      serviceAccountName: opentelemetry-sa
      containers:
        - name: api-container
          env:
            - name: ASPNETCORE_ENVIRONMENT
              value: Development
          image: [account-id].dkr.ecr.[region].amazonaws.com/opentelemetry-api:2.0
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
        - name: otel-container
          image: amazon/aws-otel-collector:latest
          env:
            - name: AWS_REGION
              value: [region]
          imagePullPolicy: Always
          command:
            - "/awscollector"
            - "--config=/conf/config.yaml"
          volumeMounts:
            - name: otel-config-volume
              mountPath: /conf/config.yaml
              subPath: config.yaml
          resources:
            limits:
              cpu: 500m
              memory: 500Mi
            requests:
              cpu: 250m
              memory: 250Mi
      volumes:
        - configMap:
            name: opentelemetry-config
          name: otel-config-volume
```

You can see all the code and scripts [here](https://github.com/raulnq/opentelemetry-sandbox/tree/eks-sidecar-otel). Thanks, and happy coding.