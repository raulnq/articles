## Running Tests with Helm


Writing tests is a key task in our day-to-day work to ensure the quality of our code. While we usually run unit tests during the build process, others types of tests, such as acceptance tests, require to have the application previously deployed in the environment. 

In a previous [post](https://blog.raulnq.com/useful-commands-for-helm), we reviewed Helm as a tool to deploy our applications into a Kubernetes cluster. Today we will explore a feature that could make our life easier when faced with this last type of testing.

We will use the default ASP.NET Core Web API template to have an application to test. In our case, the project will be called `helm-sandbox-app`. Add a `Dockerfile` to the project with the following content:

```
FROM mcr.microsoft.com/dotnet/aspnet:6.0 AS base
WORKDIR /app
EXPOSE 80

FROM mcr.microsoft.com/dotnet/sdk:6.0 AS build
WORKDIR /src
COPY ["helm-sandbox-app/helm-sandbox-app.csproj", "helm-sandbox-app/"]
RUN dotnet restore "helm-sandbox-app/helm-sandbox-app.csproj"
COPY . .
WORKDIR "/src/helm-sandbox-app"
RUN dotnet build "helm-sandbox-app.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "helm-sandbox-app.csproj" -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "helm-sandbox-app.dll"]
``` 

Next, we will create an MSTest Test project called `helm-sandbox-tests`. Add the following NuGet package to the project:

- [Microsoft.Extensions.Configuration.EnvironmentVariables](https://www.nuget.org/packages/Microsoft.Extensions.Configuration.EnvironmentVariables/6.0.1)

Modify the default test class with the following code:

```csharp
[TestClass]
public class WeatherForecastTests
{
    private readonly string _uri;

    [TestMethod]
    public async Task should_return_ok()
    {
        using (var client = new HttpClient())
        {
            client.BaseAddress = new Uri(_uri);

            var response = await client.GetAsync("WeatherForecast");

            Assert.IsNotNull(response);

            Assert.AreEqual(HttpStatusCode.OK, response.StatusCode);
        }
    }

    public WeatherForecastTests()
    {
        var config = new ConfigurationBuilder()
        .AddEnvironmentVariables()
        .Build();

        _uri = config["Uri"];
    }
}
``` 

Add a `Dockerfile` to the project with the following content:

```
FROM mcr.microsoft.com/dotnet/sdk:6.0 AS build
WORKDIR /src
COPY ["helm-sandbox-tests/helm-sandbox-tests.csproj", "helm-sandbox-tests/"]
RUN dotnet restore "helm-sandbox-tests/helm-sandbox-tests.csproj"
COPY . .
WORKDIR "/src/helm-sandbox-tests"
ENTRYPOINT ["dotnet", "test", "helm-sandbox-tests.csproj","--no-restore"]
``` 

Time to generate our local Docker images (one for your API and the other to test the API). At the solution level, run (don't forget the dot at the end):

```powershell
docker build -t raulnq/helm-sandbox-app:1.0 -f .\helm-sandbox-app\Dockerfile .
docker build -t raulnq/helm-sandbox-tests:1.0 -f .\helm-sandbox-tests\Dockerfile .
``` 

Let's create our helm package:

```powershell
mkdir helm
cd helm
helm create helm-sandbox-app
``` 

Keep only these files in your Helm package (rename the default test-connection.yaml to test.yaml):

```powershell
helm\helm-sandbox-app
|--test
|   `-- test.yaml
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
  name: {{ include "helm-sandbox-app.fullname" . }}
  labels:
    {{- include "helm-sandbox-app.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "helm-sandbox-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "helm-sandbox-app.selectorLabels" . | nindent 8 }}
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
  name: {{ include "helm-sandbox-app.fullname" . }}
  labels:
    {{- include "helm-sandbox-app.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: http
      protocol: TCP
      name: http
  selector:
    {{- include "helm-sandbox-app.selectorLabels" . | nindent 4 }}

``` 

The `test.yaml` file:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include "helm-sandbox-app.fullname" . }}-test"
  labels:
    {{- include "helm-sandbox-app.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": test-success
    "helm.sh/hook-delete-policy": before-hook-creation
spec:
  containers:
    - name: {{ .Chart.Name }}
      image: "{{ .Values.tests.repository }}:{{ .Values.tests.tag | default .Chart.AppVersion }}"
      imagePullPolicy: {{ .Values.tests.pullPolicy }}
      env:
      - name: Uri
        value: "http://{{ include "helm-sandbox-app.fullname" . }}:{{ .Values.service.port }}"
  restartPolicy: Never

``` 

And the `values.yaml` file:

```yaml
replicaCount: 1

image:
  repository: raulnq/helm-sandbox-app
  pullPolicy: IfNotPresent
  tag: "1.0"
  
tests:
  repository: raulnq/helm-sandbox-tests
  pullPolicy: IfNotPresent
  tag: "1.0"

nameOverride: ""
fullnameOverride: ""

service:
  type: ClusterIP
  port: 80
``` 

Run the following command to install your chart in your local Kubernetes cluster:

```powershell
helm upgrade release-helm-sandbox-app helm-sandbox-app --install --debug
``` 

And finally, the `helm test` command:

```powershell
helm test release-helm-sandbox-app
``` 

If everything goes fine, we will see an output like this:

```powershell
Pod release-helm-sandbox-app-test pending
Pod release-helm-sandbox-app-test pending
Pod release-helm-sandbox-app-test pending
Pod release-helm-sandbox-app-test running
Pod release-helm-sandbox-app-test succeeded
NAME: release-helm-sandbox-app
LAST DEPLOYED: Sat Jun 18 10:57:44 2022
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE:     release-helm-sandbox-app-test
Last Started:   Sat Jun 18 22:59:42 2022
Last Completed: Sat Jun 18 22:59:47 2022
Phase:          Succeeded
``` 

We can check the logs of our pod with:

```powershell
kubectl logs release-helm-sandbox-app-test
``` 

Having as a result:

```powershell
helm-sandbox-tests -> /src/helm-sandbox-tests/bin/Debug/net6.0/helm-sandbox-tests.dll
Test run for /src/helm-sandbox-tests/bin/Debug/net6.0/helm-sandbox-tests.dll (.NETCoreApp,Version=v6.0)
Microsoft (R) Test Execution Command Line Tool Version 17.2.0 (x64)
Copyright (c) Microsoft Corporation.  All rights reserved.

Starting test execution, please wait...
A total of 1 test files matched the specified pattern.

Passed!  - Failed:     0, Passed:     1, Skipped:     0, Total:     1, Duration: 115 ms - /src/helm-sandbox-tests/bin/Debug/net6.0/helm-sandbox-tests.dll (net6.0)
``` 

[Here](https://helm.sh/docs/topics/chart_tests/) you can find the official documentation about the `helm test` command. The code of this post can be found [here](https://github.com/raulnq/helm-sandbox).









