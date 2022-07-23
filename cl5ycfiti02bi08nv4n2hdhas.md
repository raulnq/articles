## Deploying AWS CloudWatch Alarms with Terraform

Today we will review how to deploy an AWS CloudWatch alarm based on a log entry using Terraform. This post is going to use as a base what we saw in [Deploying AWS Lambda Functions with Terraform](https://blog.raulnq.com/deploying-aws-lambda-functions-with-terraform) (take a look if you haven't seen it yet). Please download the content of [this](https://github.com/raulnq/aws-lambda-sandbox/tree/terraform) branch. 

Let's change first our application to randomly throw and log an error, locate the `WeatherForecastController.cs` file, and update it as follows:

```csharp
[ApiController]
[Route("[controller]")]
public class WeatherForecastController : ControllerBase
{
    private static readonly string[] Summaries = new[]
    {
        "Freezing", "Bracing", "Chilly", "Cool", "Mild", "Warm", "Balmy", "Hot", "Sweltering", "Scorching"
    };

    private readonly ILogger<WeatherForecastController> _logger;

    public WeatherForecastController(ILogger<WeatherForecastController> logger)
    {
        _logger = logger;
    }

    [HttpGet(Name = "GetWeatherForecast")]
    public IEnumerable<WeatherForecast> Get()
    {
        var throwException = Random.Shared.Next(0, 2);

        if (throwException == 0)
        {
            throw new ApplicationException("Oops!!");
        }
        return Enumerable.Range(1, 5).Select(index => new WeatherForecast
        {
            Date = DateTime.Now.AddDays(index),
            TemperatureC = Random.Shared.Next(-20, 55),
            Summary = Summaries[Random.Shared.Next(Summaries.Length)]
        })
        .ToArray();
    }
}
``` 

Create an `ExceptionFilter.cs` file with the following content:


```csharp
public class ExceptionFilter : IExceptionFilter
{
    private readonly ILogger<ExceptionFilter> _logger;

    public ExceptionFilter(ILogger<ExceptionFilter> logger) =>
        _logger = logger;

    public void OnException(ExceptionContext context)
    {
        _logger.LogError(context.Exception, context.Exception.Message);

        context.ExceptionHandled = true;

        context.Result = new ContentResult
        {
            Content = context.Exception.Message,
        };
    }
}
``` 

To generate structured logs add the [Serilog.AspNetCore](https://www.nuget.org/packages/Serilog.AspNetCore) NuGet package in the project and update the `Program.cs` file as follows:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers(options =>
{
    options.Filters.Add<ExceptionFilter>();
});
builder.Host.UseSerilog((ctx, lc) => lc
    .WriteTo.Console(new JsonFormatter()));
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();
builder.Services.AddAWSLambdaHosting(LambdaEventSource.HttpApi);
var app = builder.Build();

app.UseSwagger();
app.UseSwaggerUI();
app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();

app.Run();
``` 

Under the `terraform` folder, open the `main.tf` file. The first resource that we are going to add is the [Metric Filter](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/MonitoringLogData.html):

```json
resource "aws_cloudwatch_log_metric_filter" "error_log_metric_filter" {
  name           = "error-log-metric-filter"
  pattern        = "{ $.Level = \"Error\" }"
  log_group_name = "/aws/lambda/ASPNETCoreWebAPI"

  metric_transformation {
    name       = "ErrorCount"
    namespace  = "ASPNETCoreWebAPI"
    value      = "1"
  }
}
``` 

The [Alarm](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/AlarmThatSendsEmail.html) itself comes next:

```json
resource "aws_cloudwatch_metric_alarm" "error_alarm" {
  alarm_name                = "error-alarm"
  comparison_operator       = "GreaterThanOrEqualToThreshold"
  evaluation_periods        = "1"
  metric_name               = aws_cloudwatch_log_metric_filter.error_log_metric_filter.metric_transformation[0].name
  namespace                 = aws_cloudwatch_log_metric_filter.error_log_metric_filter.metric_transformation[0].namespace
  period                    = "300"
  statistic                 = "Sum"
  threshold                 = "1"
  alarm_description         = "UnauthorizedErrorCount >= 1"
  alarm_actions             = [aws_sns_topic.alarm_topic.arn]
  ok_actions                = [aws_sns_topic.alarm_topic.arn]
  insufficient_data_actions = [aws_sns_topic.alarm_topic.arn]
}
``` 

Now we need a way to react to the alarm. For that, we are going to create an [SNS topic](https://docs.aws.amazon.com/sns/latest/dg/welcome.html) and add a subscription to send a mail whenever the alarm is triggered:

```json
resource "aws_sns_topic" "alarm_topic" {
  name            = "alarm-topic"
  delivery_policy = jsonencode({
    "http" : {
      "defaultHealthyRetryPolicy" : {
        "minDelayTarget" : 20,
        "maxDelayTarget" : 20,
        "numRetries" : 3,
        "numMaxDelayRetries" : 0,
        "numNoDelayRetries" : 0,
        "numMinDelayRetries" : 0,
        "backoffFunction" : "linear"
      },
      "disableSubscriptionOverrides" : false,
      "defaultThrottlePolicy" : {
        "maxReceivesPerSecond" : 1
      }
    }
  })
}

resource "aws_sns_topic_subscription" "topic_email_subscription" {
  topic_arn = aws_sns_topic.alarm_topic.arn
  protocol  = "email"
  endpoint  = "raulnq@gmail.com"
}
``` 

And that's it. Time to deploy everything to AWS, go back to the solution level and run:

```powershell
mkdir terraform/publish
dotnet publish "ASPNETCoreWebAPI" --output "terraform/ASPNETCoreWebAPI" --configuration "Release" --framework "net6.0" /p:GenerateRuntimeConfigurationFiles=true --runtime linux-x64
Compress-Archive -Path terraform/publish/* -DestinationPath terraform/ASPNETCoreWebAPI.zip
cd terraform
terraform init
terraform plan -out app.tfplan
terraform apply 'app.tfplan'
``` 

After applying the terraform script, we will receive a mail to confirm the subscription:

![email.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1658606245252/CpOF1WNjw.png align="left")

Open in a browser the `lambda_url` output (from the output of the `terraform apply` command), adding `swagger` at the end. Execute the endpoint a few times until getting the error message:

![api.PNG](https://cdn.hashnode.com/res/hashnode/image/upload/v1658606522650/Qzxbj4tI6.PNG align="left")

If we look for a Log Group in CloudWatch named `/aws/lambda/ASPNETCoreWebAPI` and open one Log Stream, we will see our logs:

![logs.PNG](https://cdn.hashnode.com/res/hashnode/image/upload/v1658607017249/Np69bH8KN.PNG align="left")

And after waiting a few minutes, we got the email with our alarm:

![message.PNG](https://cdn.hashnode.com/res/hashnode/image/upload/v1658607194948/QC_aEGxYi.PNG align="left")

Clean up all the resources running:

```powershell
terraform plan -out app.tfplan -destroy
terraform apply -destroy 'app.tfplan'
``` 

You can see all the code [here](https://github.com/raulnq/aws-lambda-sandbox/tree/cloudwatch). Thanks, and happy coding.





