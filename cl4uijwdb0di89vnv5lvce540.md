## Building .NET microservices easily with Tye

Today, building microservices (technology stack aside) can be an overwhelming task due to the number of tools, technologies, and frameworks you need to learn before starting to code. One of the most widely used technology is containerization, with Docker as its best-known representative. But what does this mean for a new developer? Does he need to know how to create a docker file, a docker-compose file, or in the worst case, how to deploy your application on a local Kubernetes cluster?.

Here is where [Tye](https://github.com/dotnet/tye) comes to the rescue, helping new developers to start easily in the microservice world (until they have all the necessary knowledge). But not only them, senior developers can also see their day-to-day work improved by using Tye as a development tool.

> Tye is a developer tool that makes developing, testing, and deploying microservices and distributed applications easier. Project Tye includes a local orchestrator to make developing microservices easier and the ability to deploy microservices to Kubernetes with minimal configuration.

In this post, we will review some of the features that Tye offers thru an application to see the population of different countries.

Let's start installing Tye with the following command:

```powershell
dotnet tool install -g Microsoft.Tye --version "0.11.0-alpha.22111.1"
``` 

Now, we will create two projects, one to handle the frontend (ASP.NET Core Web App) and the other for the backend (ASP.NET Core Web API):

```powershell
dotnet new razor -n frontend
dotnet new webapi -n backend
dotnet new sln -n tye-sandbox-app
dotnet sln add frontend
dotnet sln add backend
``` 

At this point, we can execute Tye to see the default projects running:

```powershell
tye run --dashboard
``` 

The Tye run command creates the output as follows:

```powershell
[16:18:54 INF] Default dashboard port 8000 has been reserved by the application or is in use, choosing random port.
[16:18:54 INF] Executing application from C:\Source\github\tye\tye-sandbox-app.sln
[16:18:54 INF] Dashboard running on http://127.0.0.1:51711
[16:18:54 INF] Building projects
[16:18:58 INF] Application tye-sandbox-app started successfully with Pid: 103480
[16:18:58 INF] Launching service backend_ddb231fb-c: C:\Source\github\tye\backend\bin\Debug\net6.0\backend.exe
[16:18:58 INF] Launching service frontend_f9128e58-4: C:\Source\github\tye\frontend\bin\Debug\net6.0\frontend.exe
[16:18:58 INF] frontend_f9128e58-4 running on process id 94668 bound to http://localhost:51712, https://localhost:51713
[16:18:58 INF] backend_ddb231fb-c running on process id 88560 bound to http://localhost:51714, https://localhost:51715
[16:18:58 INF] Replica backend_ddb231fb-c is moving to a ready state
[16:18:58 INF] Replica frontend_f9128e58-4 is moving to a ready state
[16:18:59 INF] Selected process 94668.
[16:18:59 INF] Listening for event pipe events for frontend_f9128e58-4 on process id 94668
[16:18:59 INF] Selected process 88560.
[16:18:59 INF] Listening for event pipe events for backend_ddb231fb-c on process id 88560
``` 

As well, a browser will show the Tye dashboard:

![tye-run.PNG](https://cdn.hashnode.com/res/hashnode/image/upload/v1656192304560/U-RRKbwou.PNG align="left")

We can click on the URLs (Bindings) to see our applications running. Press `Ctrl+C` to shut down the applications. Time to customize the solution, under the `backend` project, add the `Statistics.cs` file:

```csharp
public class Statistics
{
    public string? Country { get; set; }
    public decimal Population { get; set; }
    public decimal Area { get; set; }
}
``` 

And the `StatisticsController.cs` file

```csharp
[ApiController]
[Route("[controller]")]
public class StatisticsController : ControllerBase
{
    [HttpGet]
    public IEnumerable<Statistics> Get()
    {
        return new[] {
            new Statistics() { Country = "China", Population = 1439323776, Area = 9706961 },
            new Statistics() { Country = "India", Population = 1393409038, Area = 3287590 },
            new Statistics() { Country = "United States", Population = 332915073, Area = 9372610 }
        };
    }
}
``` 

Under the `frontend` project, add `Statistics.cs` file:

```csharp
public class Statistics
{
    public string? Country { get; set; }
    public decimal Population { get; set; }
    public decimal Area { get; set; }
    public decimal Density { get { return Population / Area; } }
}
``` 

And the `StatisticsClient.cs` file:

```csharp
public class StatisticsClient
{
    private readonly HttpClient _client;

    public StatisticsClient(HttpClient client)
    {
        _client = client;
    }

    public async Task<IEnumerable<Statistics>?> Get()
    {
        var httpResponse = await _client.GetAsync("Statistics");
        var options = new JsonSerializerOptions() { PropertyNameCaseInsensitive = true };
        httpResponse.EnsureSuccessStatusCode();
        var body = await httpResponse.Content.ReadAsStringAsync();
        return JsonSerializer.Deserialize<IEnumerable<Statistics>>(body, options);
    }
}
``` 

Modify the `Index.cshtml.cs` file as follows:

```csharp
public class IndexModel : PageModel
{
    public IEnumerable<Statistics>? Statistics { get; set; }

    private readonly StatisticsClient _client;

    public IndexModel(StatisticsClient client)
    {
        _client = client;
    }

    public async Task OnGetAsync()
    {
        Statistics = await _client.Get();
    }
}
``` 

And the `Index.cshtml` file:

```html
@page
@model IndexModel
@{
    ViewData["Title"] = "Home page";
}

<h1>Statistics</h1>
<table class="table">
    <thead>
        <tr>
            <th>Country</th>
            <th>Population</th>
            <th>Area</th>
            <th>Density</th>
        </tr>
    </thead>
    <tbody>
    @if (Model.Statistics != null)
    {
        foreach (var task in Model.Statistics)
        {
            <tr>
                <td>@task.Country </td>
                <td>@task.Population</td>
                <td>@task.Area</td>
                <td>@task.Density</td>
            </tr>
        }
    }
    </tbody>
</table>
``` 

Finally, How does the `frontend` know the address of the `backend` to do the HTTP call?. Short answer, Tye injects environment variables with this information in all our applications (you can check more details about it [here](https://github.com/dotnet/tye/blob/main/docs/reference/service_discovery.md)). To make easy to use the environment variables injected by Tye, they developed a NuGet package that we are going to install:

- [Microsoft.Tye.Extensions.Configuration](https://www.nuget.org/packages/Microsoft.Tye.Extensions.Configuration/0.10.0-alpha.21420.1)

Run the `tye init` command to create a `tye.yaml` file as follows:

```yaml
name: tye-sandbox-app
services:
- name: frontend
  project: frontend/frontend.csproj
- name: backend
  project: backend/backend.csproj
```

Add the following line to `Program.cs` file (`backend` refers to the name of the `service` in the `tye.yaml` file) :

```csharp
builder.Services.AddHttpClient<StatisticsClient>(client =>
{
    client.BaseAddress = builder.Configuration.GetServiceUri("backend");
});
``` 

Run the `tye run --dashboard` and see how easy was to have two applications running and working together:

![frontend.PNG](https://cdn.hashnode.com/res/hashnode/image/upload/v1656195231646/PMqqhiWZ3.PNG align="left")

Usually, an application needs external dependencies to work, like services, databases, and others. With Tye is quite simple to add those dependencies. Let's see how to add an SQLServer database:

```yaml
name: tye-sandbox-app
services:
- name: frontend
  project: frontend/frontend.csproj
- name: backend
  project: backend/backend.csproj
- name: database
  image: mcr.microsoft.com/mssql/server:2017-latest
  env:
  - name: ACCEPT_EULA
    value: "Y"
  - name: SA_PASSWORD
    value: "pass@word1"
  volumes:
    - source: mssql/
      target: /var/opt/mssql
  bindings:
  - port: 1433
    connectionString: Server=${host},${port};Database=StatisticsDb;User ID=sa;Password=${env:SA_PASSWORD};MultipleActiveResultSets=true;
``` 

We are adding a new section called `database` and defining the image that instructs Tye to pull the Docker image when the `tye run` command is issued. Create an `mssql` folder:

```powershell
mkdir mssql
``` 

Add the following Nuget packages to the `backend` project:

- [Dapper](https://www.nuget.org/packages/Dapper/)
- [System.Data.SqlClient](https://www.nuget.org/packages/System.Data.SqlClient)

Modify the ` StatisticsController.cs` file as follows:

```csharp
[ApiController]
[Route("[controller]")]
public class StatisticsController : ControllerBase
{
    private readonly string _connectionString;
    public StatisticsController(IConfiguration configuration)
    {
        _connectionString = configuration.GetConnectionString("database");
    }

    [HttpGet]
    public async Task<IEnumerable<Statistics>> Get()
    {
        using (var con = new SqlConnection(_connectionString))
        {
            return await (con.QueryAsync<Statistics>("SELECT * FROM [Statistics]"));
        }
    }
}
``` 

Run the `tye run --dashboard` command. Use your favorite SQLServer client to run the following script (SqlServer could take a couple of minutes to be ready to accept connections):

```sql
USE [master]
GO
CREATE DATABASE [StatisticsDb]
GO
USE [StatisticsDb]
GO
CREATE TABLE [Statistics](
    [Id] int IDENTITY(1,1) NOT NULL,
    [Country] varchar(255) NOT NULL,
    [Population] decimal(15) NOT NULL,
	[Area] decimal(19,4) NOT NULL
    CONSTRAINT [PK_Statistics] PRIMARY KEY ([Id])
)
INSERT INTO [Statistics](Country, Population, Area) VALUES ('China',1448471400, 9706961)
INSERT INTO [Statistics](Country, Population, Area) VALUES ('India',1393409038, 3287590)
INSERT INTO [Statistics](Country, Population, Area) VALUES ('United States', 332915073, 9372610)
``` 

Go to the frontend URL to see the application running. Tye offers integration with popular tools; these features are called extensions. [Here](https://github.com/dotnet/tye/blob/main/docs/recipes/logging_seq.md), we can see how to use [Seq](https://datalust.co/seq) to view all the logs produced by our application (with zero code), let's modify the again the `tye.yaml` file:

```yaml
name: tye-sandbox-app
services:
- name: frontend
  project: frontend/frontend.csproj
- name: backend
  project: backend/backend.csproj
- name: database
  image: mcr.microsoft.com/mssql/server:2017-latest
  env:
  - name: ACCEPT_EULA
    value: "Y"
  - name: SA_PASSWORD
    value: "pass@word1"
  volumes:
    - source: mssql/
      target: /var/opt/mssql
  bindings:
  - port: 1433
    connectionString: Server=${host},${port};Database=StatisticsDb;User ID=sa;Password=${env:SA_PASSWORD};MultipleActiveResultSets=true;
extensions:
- name: seq
  logPath: ./.logs

``` 
Run the `tye run --dashboard` command:

![extensions.PNG](https://cdn.hashnode.com/res/hashnode/image/upload/v1656197471727/vfjZNxB8c.PNG align="left")

And go to the Seq dashboard:

![seq.PNG](https://cdn.hashnode.com/res/hashnode/image/upload/v1656197549881/sa0Auv7at.PNG align="left")

Tye is still in the experimental phase, but the current feature can speed up your day-to-day coding. [Here](https://github.com/raulnq/tye-sandbox) you can find all the code used in this post.







