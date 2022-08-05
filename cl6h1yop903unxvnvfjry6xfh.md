## OpenTelemetry: The OTEL Collector

In the post [Observability with OpenTelemetry in .NET](https://blog.raulnq.com/observability-with-opentelemetry-in-net), we saw how OpenTelemetry tries to provide a standardized way to instrument our applications. OpenTelemetry achieves that by providing us with a specification that can be implemented in multiple languages. But there is another component, the [OTEL Collector](https://opentelemetry.io/docs/collector/):

> The OpenTelemetry Collector offers a vendor-agnostic implementation of how to receive, process and export telemetry data. It removes the need to run, operate, and maintain multiple agents/collectors. This works with improved scalability and supports open-source observability data formats (e.g. Jaeger, Prometheus, Fluent Bit, etc.) sending to one or more open-source or commercial back-ends. The local Collector agent is the default location to which instrumentation libraries export their telemetry data.

In short, instead of sending our telemetry data to a specific backend, we could send it to OTEL Collector and from there to any backend. Additionally, the collector can take care of additional handling like retries, batching, encryption, or even sensitive data filtering. The OTEL Collector consists of three components that access telemetry data: **receivers**, **processors**, and **exporters**, which are used to build **pipelines**.

### Receivers

> A receiver is how data gets into the OpenTelemetry Collector. Generally, a receiver accepts data in a specified format, translates it into the internal format and passes it to processors and exporters defined in the applicable pipelines.

[Here](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver), you can find all the available receivers.

### Exporters

> An exporter is how data gets sent to different systems/back-ends. Generally, an exporter translates the internal format into another defined format.

[Here](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter), you can find all the available exporters.

### Processors

> Processors are used at various stages of a pipeline. Generally, a processor pre-processes data before it is exported (e.g. modify attributes or sample) or helps ensure that data makes it through a pipeline successfully (e.g. batch/retry).

### Pipelines

> A pipeline consists of a set of receivers, processors and exporters. Each receiver/processor/exporter must be defined in the configuration outside of the service section to be included in a pipeline. Pipelines can be of the following types:
- Traces: collects and processes trace data.
- Metrics: collects and processes metric data.
- Logs: collects and processes log data.

Let's clone or download the following [code](https://github.com/raulnq/opentelemetry-sandbox/tree/main). Open the solution and add the following NuGet packages:

- [OpenTelemetry.Exporter.OpenTelemetryProtocol](https://www.nuget.org/packages/OpenTelemetry.Exporter.OpenTelemetryProtocol/1.3.0)
- [OpenTelemetry.Exporter.OpenTelemetryProtocol.Logs](https://www.nuget.org/packages/OpenTelemetry.Exporter.OpenTelemetryProtocol.Logs/1.3.0-rc.2)

Edit the `Program.cs` file as follow:

```csharp
using OpenTelemetry.Resources;
using OpenTelemetry.Trace;
using OpenTelemetry.Metrics;
using OpenTelemetry.Logs;
using AnimeQuoteApi;

var builder = WebApplication.CreateBuilder(args);

// Add services to the container.
builder.Services.AddControllers();
// Learn more about configuring Swagger/OpenAPI at https://aka.ms/aspnetcore/swashbuckle
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();
builder.Services.AddHttpClient<QuoteClient>(client=>
{
    client.BaseAddress = new Uri("https://animechan.vercel.app");
});

builder.Services.AddOpenTelemetryTracing(builder =>
{
    builder
    .AddOtlpExporter()
    .AddSource("AnimeQuoteApi")
    .SetResourceBuilder(ResourceBuilder.CreateDefault().AddService("AnimeQuoteApi"))
    .AddHttpClientInstrumentation()
    .AddAspNetCoreInstrumentation()
    ;
});

builder.Services.AddOpenTelemetryMetrics(builder =>
{
    builder
    .AddRuntimeInstrumentation()
    .AddAspNetCoreInstrumentation()
    .AddHttpClientInstrumentation()
    .SetResourceBuilder(ResourceBuilder.CreateDefault().AddService("AnimeQuoteApi"))
    .AddOtlpExporter();
});

builder.Logging.ClearProviders();
builder.Logging.AddOpenTelemetry(builder =>
{
    builder
    .SetResourceBuilder(ResourceBuilder.CreateDefault().AddService("AnimeQuoteApi"))
    .AddOtlpExporter();
});

var app = builder.Build();

// Configure the HTTP request pipeline.
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();

app.UseAuthorization();

app.MapControllers();

app.Run();
``` 

Run the following command to start [Jeager](https://www.jaegertracing.io/docs/1.36/):

```powershell
docker run -d --name jaeger -e COLLECTOR_ZIPKIN_HOST_PORT=:9411 -e COLLECTOR_OTLP_ENABLED=false -p 6831:6831/udp -p 6832:6832/udp -p 5778:5778 -p 16686:16686 -p 14250:14250 -p 14268:14268 -p 14269:14269 -p 9411:9411  jaegertracing/all-in-one:1.36
``` 

Edit the `prometheus.yml` file as follows:

```yaml
global:
  scrape_interval: 10s
  evaluation_interval: 10s

scrape_configs:
  - job_name: "AnimeQuoteApi"
    scrape_interval: 5s
    static_configs:
      - targets: ["host.docker.internal:8889"]
``` 

Run the following command to start [Prometheus](https://prometheus.io/docs/prometheus/latest/installation/):

```powershell
docker run -d --name prometheus -p 9090:9090 -v ${pwd}/prometheus.yml:/etc/prometheus/prometheus.yml prom/prometheus
``` 

Add an `otel-collector-config.yaml` file with the following content:

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 15s
    send_batch_size: 10

exporters:
  jaeger:
    endpoint: "host.docker.internal:14250"
    tls:
      insecure: true
  prometheus:
    endpoint: "0.0.0.0:8889"
    namespace: "default"
  logging:
    loglevel: debug

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [jaeger]
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [prometheus]
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [logging]
``` 
 Add a `docker-compose.yml` file:

```yaml
version: "3.4"

services:
  aws-ot-collector:
    image: otel/opentelemetry-collector-contrib:0.56.0
    command: ["--config=/etc/otel-collector-config.yaml"]
    volumes:
      - ./otel-collector-config.yaml:/etc/otel-collector-config.yaml
    ports:
      - "8888:8888"   # Prometheus metrics exposed by the collector
      - "8889:8889"   # Prometheus exporter metrics
      - "13133:13133" # health_check extension
      - "4317:4317"   # OTLP gRPC receiver
      - "4318:4318"   # OTLP http receiver
``` 

And, run the following command to start the [OTEL Collector](https://opentelemetry.io/docs/collector/getting-started/):

```
docker-compose up
``` 

Run the application and send a couple of requests to start to see traces in Jeager (http://localhost:16686/search):

![jeager.PNG](https://cdn.hashnode.com/res/hashnode/image/upload/v1659739026825/2_b_qriQi.PNG align="left")

Metrics in Prometheus (http://localhost:9090/):

![prometheus.PNG](https://cdn.hashnode.com/res/hashnode/image/upload/v1659739075450/aj0f-QX2R.PNG align="left")

And logs in the console:

![console.PNG](https://cdn.hashnode.com/res/hashnode/image/upload/v1659739111010/p3og_3fok.PNG align="left")

You can see all the code [here](https://github.com/raulnq/opentelemetry-sandbox/tree/otel). Thanks, and happy coding.