## Observability with OpenTelemetry in .NET

Over the past few years, the transition from monolithic applications to microservices (despite their benefits under the right circumstances) has brought a new set of challenges. One of them is **Observability**:

> Observability lets us understand a system from the outside, by letting us ask questions about that system without knowing its inner workings. 

Observability is essential to ensure that an application and its dependencies behave as expected and to identify and speed the resolution of any issues that may arise. To that end, teams need to gather telemetry data (metrics, logs, and traces) from all those parts. There are several observability tools out there trying to solve the problem (or part of it), resulting in no standardized way to do it.

> With a lack of standardization, the net result is the lack of data portability and the burden on the user to maintain instrumentation libraries.

Under this context, [OpenTelemetry](https://opentelemetry.io/docs/) comes as a unified standard for service instrumentation. It provides us with:

- A single, vendor-agnostic instrumentation library per language with support for both automatic and manual instrumentation.
- An end-to-end implementation to generate, emit, collect, process and export telemetry data.
- And more...

Today we will review three categories of telemetry data supported by OpenTelemetry and how to use it with [.NET](https://github.com/open-telemetry/opentelemetry-dotnet):

- **Traces**: Traces give us the big picture of what happens when a request is made by user or an application.
- **Metrics**: A metric is a measurement about a service, captured at runtime.
- **Logs**: A log is a timestamped text record, either structured (recommended) or unstructured, with metadata.

We will develop an application to get a random quote from an anime character (using the [Animechan API](https://animechan.vercel.app/)). Let's create the project and solution:

```powershell
dotnet new webapi -n AnimeQuoteApi
dotnet new sln -n openteletry-sandbox
dotnet sln add AnimeQuoteApi
``` 

Add a `Quote.cs` file to deserialize the response from the API:

```csharp
public class Quote
{
    [JsonPropertyName("anime")]
    public string? Anime { get; set; }
    [JsonPropertyName("character")]
    public string? Character { get; set; }
    [JsonPropertyName("quote")]
    public string? Text { get; set; }
}
``` 

Add a `QuoteClient.cs` to call the API:

```csharp
public class QuoteClient
{
    private readonly HttpClient _httpClient;

    public QuoteClient(HttpClient httpClient)
    {
        _httpClient = httpClient;
    }
    public async Task<Quote> Get()
    {
        var httpResponse = await _httpClient.GetAsync("/api/random");

        httpResponse.EnsureSuccessStatusCode();

        var body = JsonSerializer.Deserialize<Quote>(await httpResponse.Content.ReadAsStringAsync());

        return body!;
    }
}
``` 

And add a `QuotesController.cs` file:

```csharp
[ApiController]
[Route("[controller]")]
public class QuotesController : ControllerBase
{
    private readonly QuoteClient _client;

    public QuotesController(QuoteClient client)
    {
        _client = client;
    }

    [HttpGet()]
    public Task<Quote> Get()
    {
        return _client.Get();
    }
}
``` 

To end, add these lines to the `Program.cs` file:

```csharp
builder.Services.AddHttpClient<QuoteClient>(client=>
{
    client.BaseAddress = new Uri("https://animechan.vercel.app");
});
``` 

### Traces

Let's start adding a few NuGet packages:

- [OpenTelemetry](https://www.nuget.org/packages/OpenTelemetry)
- [OpenTelemetry.Extensions.Hosting](https://www.nuget.org/packages/OpenTelemetry.Extensions.Hosting/1.0.0-rc9.4)
- [OpenTelemetry.Instrumentation.AspNetCore](https://www.nuget.org/packages/OpenTelemetry.Instrumentation.AspNetCore/1.0.0-rc9.4)
- [OpenTelemetry.Instrumentation.Http](https://www.nuget.org/packages/OpenTelemetry.Instrumentation.Http/1.0.0-rc9.4)
- [OpenTelemetry.Exporter.Jaeger](https://www.nuget.org/packages/OpenTelemetry.Exporter.Jaeger)

To visualize and analyze your traces and metrics, you will need to export them to a backend. We will use [Jeager](https://www.jaegertracing.io/docs/1.36/). Run the following command to start it locally:

```powershell
docker run -d --name jaeger -e COLLECTOR_ZIPKIN_HOST_PORT=:9411 -e COLLECTOR_OTLP_ENABLED=true -p 6831:6831/udp -p 6832:6832/udp -p 5778:5778 -p 16686:16686 -p 4317:4317 -p 4318:4318 -p 14250:14250 -p 14268:14268 -p 14269:14269 -p 9411:9411  jaegertracing/all-in-one:1.36
``` 

Add this code to the `Program.cs` file:

```csharp
builder.Services.AddOpenTelemetryTracing(builder =>
{
    builder
    .AddJaegerExporter()
    .AddSource("AnimeQuoteApi")
    .SetResourceBuilder(ResourceBuilder.CreateDefault().AddService("AnimeQuoteApi"))
    .AddHttpClientInstrumentation()
    .AddAspNetCoreInstrumentation()
    ;
});
``` 

Run the application and send a couple of requests to see our traces in Jaeger (http://localhost:16686/search):

![jaeger.PNG](https://cdn.hashnode.com/res/hashnode/image/upload/v1659276527489/HoaBTe1Pw.PNG align="left")

### Metrics

Add the following NuGet packages:

- [OpenTelemetry.Instrumentation.Runtime](https://www.nuget.org/packages/OpenTelemetry.Instrumentation.Runtime/1.0.0-rc.3) 
- [OpenTelemetry.Exporter.Prometheus](https://www.nuget.org/packages/OpenTelemetry.Exporter.Prometheus/1.3.0-rc.2) 

And the following code to the `Program.cs` file:

```csharp
builder.Services.AddOpenTelemetryMetrics(builder =>
{
    builder
    .AddRuntimeInstrumentation()
    .AddAspNetCoreInstrumentation()
    .AddHttpClientInstrumentation()
    .SetResourceBuilder(ResourceBuilder.CreateDefault().AddService("AnimeQuoteApi"))
    .AddPrometheusExporter(options =>
    {
        options.StartHttpListener = true;
        options.HttpListenerPrefixes = new string[] {  "http://127.0.0.1:9464/" };
        options.ScrapeResponseCacheDurationMilliseconds = 0;
    });
});
``` 

We will use [Prometheus](https://prometheus.io/docs/introduction/overview/) to visualize our metrics locally. First, we need to create a `prometheus.yml` file with the following content:

```yaml
global:
  scrape_interval: 10s
  evaluation_interval: 10s

scrape_configs:
  - job_name: "AnimeQuoteApi"
    scrape_interval: 5s
    static_configs:
      - targets: ["host.docker.internal:9464"]
``` 

And startup Prometheus with the following command:

```powershell
docker run -d --name prometheus -p 9090:9090 -v ${pwd}/prometheus.yml:/etc/prometheus/prometheus.yml prom/prometheus
``` 

Run the application and send a couple of requests to see our metrics in Prometheus (http://localhost:9090/graph):

![prometheus.PNG](https://cdn.hashnode.com/res/hashnode/image/upload/v1659277431808/eQ3H8Qje6.PNG align="left")

### Logs

We will use the console to visualize the logs. Add the following NuGet package:

- [OpenTelemetry.Exporter.Console](https://www.nuget.org/packages/OpenTelemetry.Exporter.Console) 

And the following code to the `Program.cs` file:

```csharp
builder.Logging.ClearProviders();
builder.Logging.AddOpenTelemetry(builder =>
{
    builder
    .SetResourceBuilder(ResourceBuilder.CreateDefault().AddService("AnimeQuoteApi"))
    .AddConsoleExporter();
});
``` 

Run the application and send a couple of requests to see our logs in the console:

![console.PNG](https://cdn.hashnode.com/res/hashnode/image/upload/v1659279068099/vCRFs9pj1.PNG align="left")

OpenTelemetry is growing and could fit well in your projects if you are developing distributed systems. You can see all the code [here](https://github.com/raulnq/opentelemetry-sandbox). Thanks, and happy coding.





