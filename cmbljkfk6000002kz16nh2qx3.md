---
title: "How to Integrate Google Logging into .NET Applications"
datePublished: Sat Jun 07 2025 01:14:31 GMT+0000 (Coordinated Universal Time)
cuid: cmbljkfk6000002kz16nh2qx3
slug: how-to-integrate-google-logging-into-net-applications
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1749246584766/10b5f860-e46b-49d7-a49a-cb5c83772b50.png
tags: logging, dotnet, gcp

---

Are you looking for cloud logging services that offer a generous free tier, support structured logs, and easily integrate with .NET? [Google Cloud Logging](https://cloud.google.com/logging/docs/overview) is a strong candidate if you're open to using the [Google Cloud Platform (GCP)](https://cloud.google.com/?hl=en) ecosystem. The service provides 50 GiB of log ingestion for free each month and offers a 30-day retention period at no extra cost (check the [official pricing page](https://www.google.com/url?sa=E&q=https%3A%2F%2Fcloud.google.com%2Fstackdriver%2Fpricing)).

> Google Cloud Logging is a fully managed, real-time log management service that allows us to store, search, analyze, monitor, and set alerts for our application's log data.

Let's build a simple .NET 9 Web API that sends its logs to Google Cloud Logging.

## **Pre-requisites**

* A [Google Cloud Project](https://developers.google.com/workspace/guides/create-project).
    
* [Cloud Logging API](https://cloud.google.com/logging/docs/api/enable-api) activated.
    

## Creating the Web API

Run the following commands:

```powershell
dotnet new webapi -n MyApi
dotnet add ./MyApi package Serilog.AspNetCore
dotnet add ./MyApi package Serilog.Sinks.GoogleCloudLogging
```

Open the `Program.cs` file and replace its contents with the following:

```csharp
using Microsoft.AspNetCore.Mvc;
using Serilog;
using Serilog.Sinks.GoogleCloudLogging;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddOpenApi();

builder.Host.UseSerilog((context, services, configuration) => configuration
    .ReadFrom.Configuration(context.Configuration)
    .ReadFrom.Services(services)
    .Enrich.FromLogContext()
    .WriteTo.Console()
    .WriteTo.GoogleCloudLogging(new GoogleCloudLoggingSinkOptions
    {
        ProjectId = "<MY_PROJECT_ID>",
        UseSourceContextAsLogName = true,
    })
);

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();
}

app.UseHttpsRedirection();

var summaries = new[]
{
    "Freezing", "Bracing", "Chilly", "Cool", "Mild", "Warm", "Balmy", "Hot", "Sweltering", "Scorching"
};

app.MapGet("/weatherforecast", ([FromServices] ILogger<Program> logger) =>
{
    logger.LogInformation("weatherforecast started");

    var forecast = Enumerable.Range(1, 5).Select(index =>
        new WeatherForecast
        (
            DateOnly.FromDateTime(DateTime.Now.AddDays(index)),
            Random.Shared.Next(-20, 55),
            summaries[Random.Shared.Next(summaries.Length)]
        ))
        .ToArray();
    return forecast;
})
.WithName("GetWeatherForecast");

app.Run();

record WeatherForecast(DateOnly Date, int TemperatureC, string? Summary)
{
    public int TemperatureF => 32 + (int)(TemperatureC / 0.5556);
}
```

The [Google Cloud Logging Serilog sink](https://github.com/manigandham/serilog-sinks-googlecloudlogging) is the easiest way to send our logs.

## Understanding the Authorization Mechanism

### Hosted Inside GCP

When our application runs on a GCP service such as Cloud Run or GKE, authorization happens automatically. The environment supplies credentials from an attached Service Account. We need to make sure that the service account has the `Logs Writer` role.

### Hosted Outside GCP

When running locally, we need to use a [Service Account key file](https://cloud.google.com/iam/docs/keys-create-delete):

* Create a Service Account: GCP Console -&gt; `IAM & Admin` -&gt; `Service Accounts` -&gt; `Create Service Account`.
    
* Grant it the `Logs Writer` role.
    
* Create a JSON key for this service account and download it.
    
* Set the environment variable `GOOGLE_APPLICATION_CREDENTIALS` to the path of this downloaded JSON key file.
    

```csharp
$env:GOOGLE_APPLICATION_CREDENTIALS="C:\path\to\your\keyfile.json"
```

Since we are running it locally, we will place the `keyfile.json` file at the project level and add the environment variable to the `launchSettings.json`:

```csharp
{
  "$schema": "https://json.schemastore.org/launchsettings.json",
  "profiles": {
    "http": {
      "commandName": "Project",
      "dotnetRunMessages": true,
      "launchBrowser": false,
      "applicationUrl": "http://localhost:5093",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development",
        "GOOGLE_APPLICATION_CREDENTIALS": "keyfile.json"
      }
    },
    "https": {
      "commandName": "Project",
      "dotnetRunMessages": true,
      "launchBrowser": false,
      "applicationUrl": "https://localhost:7171;http://localhost:5093",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
        "GOOGLE_APPLICATION_CREDENTIALS": "keyfile.json"
      }
    }
  }
}
```

Run the application and start invoking the endpoint.

> Never commit the service account key file to your source code repository.

## Viewing our Logs

Go to the GCP Console and navigate to `Logging` -&gt; `Logs Explorer`. Use the following query to filter our logs:

```sql
logName: "projects/MY_PROJECT_ID/logs"
```

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1749254437790/03f8fc34-8b1f-4ad2-9285-adf97abff5f4.png align="center")

You can find all the code [here](https://github.com/raulnq/google-logging-api-net). Thanks, and happy coding.