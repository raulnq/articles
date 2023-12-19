---
title: ".NET and Kubernetes: Collect dumps on crash"
datePublished: Tue Dec 19 2023 01:45:36 GMT+0000 (Coordinated Universal Time)
cuid: clqbontlg000008l7gvj19nvp
slug: net-and-kubernetes-collect-dumps-on-crash
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1702654834002/9777f9d8-d103-4d86-aeb0-1daacc31e23f.png
tags: kubernetes, net, crash

---

Collecting dumps after a crash is an invaluable practice for understanding why it occurred. For .NET applications, this can be achieved simply by [specifying environment variables](https://learn.microsoft.com/en-us/dotnet/core/diagnostics/collect-dumps-crash). However, how do we accomplish this when hosting our application in a Kubernetes cluster? It's particularly challenging when the dump is generated inside the container, which will be destroyed shortly after the crash. In this article, we will explore how [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/) can help address this challenge.

## Pre-requisites

* Install [Docker Desktop](https://docs.docker.com/desktop/install/windows-install/)
    
* Enable [Kubernetes](https://docs.docker.com/desktop/kubernetes/) (the standalone version included in Docker Desktop)
    

## The Application

Run the following command to create a simple .NET API:

```powershell
dotnet new web -o CrashApi
dotnet new sln -n CrashApi
dotnet sln add --in-root CrashApi
```

Navigate to the `Program.cs` file and update it as follows:

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();
app.MapGet("/", () => "Hello World!");
throw new Exception("Boom!!");
app.Run();
```

This application will crash as soon as it starts.

## **The Container Image**

At the project level, we need a `Dockerfile` to containerize our application. It should include the following content:

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:7.0 AS base
WORKDIR /app

FROM mcr.microsoft.com/dotnet/sdk:7.0 AS build
COPY ["CrashApi/CrashApi.csproj", "CrashApi/"]
RUN dotnet restore "CrashApi/CrashApi.csproj"
COPY . .
WORKDIR "/CrashApi"
RUN dotnet build "CrashApi.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "CrashApi.csproj" -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "CrashApi.dll"]
```

At the solution level, execute the following command to create the container image:

```powershell
docker build -t raulnq/crashapi:1.0 -f .\CrashApi\Dockerfile .
```

## The Persistent Volume

In the persistent volume, we will mount the local directory `C:\Temp`. Create a `persistentvolume.yaml` file at the solution level with the following content:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-persistent-volume
  labels:
    type: local
spec:
  storageClassName: hostpath
  capacity:
    storage: 256Mi
  accessModes:
    - ReadWriteMany
  hostPath:
    path: "/run/desktop/mnt/host/c/Temp"
  persistentVolumeReclaimPolicy: Retain
```

Since we are using Docker Desktop for Windows, the `storageClassName` is set to [`hostpath`](https://kubernetes.io/docs/concepts/storage/volumes/#hostpath), and the `path` is configured as `/run/desktop/mnt/host/c/Temp`. Generally, the path follows the pattern `/run/desktop/mnt/host/MY_LOCAL_PATH`. To create the Persistent Volume Claim, create a `persistentvolumeclaim.yaml` file with the following content:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-persistent-volume-claim
spec:
  storageClassName: hostpath
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 128Mi
```

## The Deployment

Create a `deployment.yaml` file with the following content:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployments
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demo
  template:
    metadata:
      labels:
        app: demo
    spec:
      containers:
      - name: my-container
        image: raulnq/crashapi:1.0
        imagePullPolicy: IfNotPresent
        env:
        - name: DOTNET_DbgEnableMiniDump
          value: "1"
        - name: DOTNET_DbgMiniDumpType
          value: "1"
        - name: DOTNET_DbgMiniDumpName
          value: "/dumps/coredump.%p.dmp"
        volumeMounts:
        - mountPath: /dumps
          name: my-volume
      volumes:
      - name: my-volume
        persistentVolumeClaim:
          claimName: my-persistent-volume-claim
```

Here, we mounted the previously created Persistent Volume Claim and set the following environment variables:

* `COMPlus_DbgEnableMiniDump` or `DOTNET_DbgEnableMiniDump`: If set to 1, this enables core dump generation.
    
* `COMPlus_DbgMiniDumpType` or `DOTNET_DbgMiniDumpType`: Specifies the type of dump to be collected (`Mini`, `Heap`, `Triage`, and `Full`).
    
* `COMPlus_DbgMiniDumpName` or `DOTNET_DbgMiniDumpName`: Path to the file where the dump will be written. The `%p` acts as a wildcard for the PID of the dumped process.
    

The prefix `DOTNET_` becomes functional, starting with .NET 7 on all platforms. Execute the following commands to deploy the application to the cluster:

```powershell
kubectl apply -f .\persistentvolume.yaml
kubectl apply -f .\persistentvolumeclaim.yaml
kubectl apply -f .\deployment.yaml
```

## The Dump

The command `kubectl get pods` will display our application:

```powershell
NAME                              READY   STATUS             RESTARTS       AGE
my-deployments-5884c58f88-mgxvw   0/1     CrashLoopBackOff   4 (84s ago)    2m59s
```

Check the pod logs with the command `kubectl logs my-deployments-5884c58f88-mgxvw`:

```powershell
Unhandled exception. System.Exception: Boom!!
   at Program.<Main>$(String[] args) in /CrashApi/Program.cs:line 4
```

To analyze the dump, we need a tool like [`dotnet-dump`](https://learn.microsoft.com/es-es/dotnet/core/diagnostics/dotnet-dump). Run the following command to install it:

```powershell
dotnet tool install --global dotnet-dump
```

Finally, run `dotnet-dump analyze C:\Temp\coredump.1.dmp` to begin the analysis (you might need to delete our deployment first):

```powershell
Loading core dump: C:\Temp\coredump.1.dmp ...
Ready to process analysis commands. Type 'help' to list available commands or 'help [command]' to get detailed help on a command.
Type 'quit' or 'exit' to exit the session.
```

As a final note, in a real environment, the persistent volume should be replaced with an appropriate type like [`csi`](https://kubernetes.io/docs/concepts/storage/volumes/#csi). The final code can be found [here](https://github.com/raulnq/k8s-dumps-crash)**.** Thank you, and happy coding.