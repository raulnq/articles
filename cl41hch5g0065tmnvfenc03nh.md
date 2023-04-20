---
title: "Nuke: Deploy Helm package locally (special guest, GitVersion)"
datePublished: Sun Jun 05 2022 15:49:37 GMT+0000 (Coordinated Universal Time)
cuid: cl41hch5g0065tmnvfenc03nh
slug: nuke-deploy-helm-package-locally-special-guest-gitversion
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1654435955345/hJjRLqXjS.png
tags: kubernetes, git, helm, nuke, gitversion

---

Today we are going to continue the journey with [Nuke](https://nuke.build/), adding new targets to the solution presented in [Nuke: Deploy ASP. NET Web App to Azure](https://blog.raulnq.com/nuke-deploy-asp-net-web-app-to-azure), to deploy our app to a local [Kubernetes](https://kubernetes.io/) cluster using [Helm](https://helm.sh/).

But first, you are going to need the following:

* A standalone [Kubernetes server and client](https://docs.docker.com/desktop/kubernetes/).
    
* [Helm CLI](https://helm.sh/docs/intro/install/).
    

### Generating the build number

Our first target will get the build number using GitVersion (we talked about it in [Semantic Versioning with GitVersion](https://blog.raulnq.com/semantic-versioning-with-gitversion-gitflow)). Let's start adding the following Nuget package:

* [GitVersion.CommandLine](https://www.nuget.org/packages/GitVersion.CommandLine/)
    

Then, add the following namespaces to get access to all `gitversion` commands:

```csharp
using static Nuke.Common.Tools.GitVersion.GitVersionTasks;
using Nuke.Common.Tools.GitVersion;
```

Add a variable to hold the generated build number:

```csharp
private string BuildNumber;
```

And the target itself:

```csharp
Target GetBuildNumber => _ => _
    .Executes(() =>
    {
        var (result, _) = GitVersion();
        BuildNumber = result.SemVer;
    });
```

### Building the Docker image

The first step is to create a `Dockerfile` in our project folder as follows:

```bash
FROM mcr.microsoft.com/dotnet/sdk:6.0 AS build-env
WORKDIR /app

# Copy everything
COPY . ./
# Restore as distinct layers
RUN dotnet restore "./nuke-sandbox-app/nuke-sandbox-app.csproj"
# Build and publish a release
RUN dotnet publish "./nuke-sandbox-app/nuke-sandbox-app.csproj" -c Release -o out

# Build runtime image
FROM mcr.microsoft.com/dotnet/aspnet:6.0
WORKDIR /app
COPY --from=build-env /app/out .
ENTRYPOINT ["dotnet", "nuke-sandbox-app.dll"]
```

Add the following namespaces to get access to all `docker` commands:

```csharp
using static Nuke.Common.Tools.Docker.DockerTasks;
using Nuke.Common.Tools.Docker;
```

Add a variable to contain the Docker image repository used in the project:

```csharp
private readonly string Repository = "raulnq/nuke-sandbox-app";
```

And as the last step, add the target:

```csharp
Target BuildImage => _ => _
    .DependsOn(GetBuildNumber)
    .Executes(() =>
    {
        var dockerFile = RootDirectory / "nuke-sandbox-app" / "Dockerfile";
        var image = $"{Repository}:{BuildNumber}";

        DockerBuild(s => s
            .SetPath(RootDirectory)
            .SetFile(dockerFile)
            .SetTag(image)
            );
    });
```

### Installing the Helm package

And finally, the step to install the Helm package in the Kubernetes cluster (you can first check [Useful commands for Helm](https://blog.raulnq.com/useful-commands-for-helm)). Go to your solution folder and run the following commands to create the Helm package:

```powershell
mkdir helm
cd helm
helm create nuke-sandbox-app
```

Keep only these files in your Helm package:

```powershell
helm\nuke-sandbox-app
|-- templates
|   |-- _helpers.tpl
|   |-- deployment.yaml
|   `-- service.yaml
|-- .helmignore
|-- Chart.yaml
`-- values.yaml
```

Modify the `deployment.yaml` file as follows:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "nuke-sandbox-app.fullname" . }}
  labels:
    {{- include "nuke-sandbox-app.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "nuke-sandbox-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "nuke-sandbox-app.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
```

The `service.yaml` file:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "nuke-sandbox-app.fullname" . }}
  labels:
    {{- include "nuke-sandbox-app.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: http
      protocol: TCP
      name: http
  selector:
    {{- include "nuke-sandbox-app.selectorLabels" . | nindent 4 }}
```

And the `values.yaml` file:

```yaml
replicaCount: 1

image:
  repository: raulnq/nuke-sandbox-app
  pullPolicy: IfNotPresent
  tag: "1.0"

nameOverride: ""
fullnameOverride: ""

service:
  type: ClusterIP
  port: 80
```

Go back to the `build.cs` file and add:

```csharp
using static Nuke.Common.Tools.Helm.HelmTasks;
using Nuke.Common.Tools.Helm;
using System.Collections.Generic;
```

Add a variable to contain the Helm release name used in the project:

```csharp
private readonly string ReleaseName = "release-nuke-sandbox-app";
```

To end, we are going to add two targets, one to install and the other to uninstall the Helm package:

```csharp
Target HelmInstall => _ => _
    .DependsOn(BuildImage)
    .Executes(() =>
    {
        var chart = HelmDirectory / "nuke-sandbox-app";

        HelmUpgrade(s => s
            .SetRelease(ReleaseName)
            .SetSet(new Dictionary<string, object>() { { "image.tag", BuildNumber }, { "image.repository", Repository } })
            .SetChart(chart)
            .EnableInstall()
        );
    });

Target HelmUninstall => _ => _
    .Executes(() =>
    {
        HelmDelete(s => s
        .SetReleaseNames(ReleaseName)
        );
    });
```

That's it, time to run our Nuke command:

```powershell
nuke HelmInstall
```

We can check our deployment with `kubectl get deployments`:

```powershell
NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
release-nuke-sandbox-app   1/1     1            1           18s
```

And uninstall when needed:

```powershell
nuke HelmUninstall
```

You can find the updated solution [here](https://github.com/raulnq/nuke-sandbox/tree/helm).