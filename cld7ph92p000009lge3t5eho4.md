# Reducing boilerplate code with Lambda Annotations

After some time developing Lambda functions with .NET one can notice that the programming model could be considered low-level, especially if we compare it with ASP.NET Core. Let's check the following code:

```csharp
public async Task<APIGatewayHttpApiV2ProxyResponse> RegisterNews(APIGatewayHttpApiV2ProxyRequest request, ILambdaContext context)
    {
        try
        {
            var body = System.Text.Json.JsonSerializer.Deserialize<RegisterNewsRequest>(request.Body)!;

            var news = new News() { Id = Guid.NewGuid(), Description = body.Description, Title = body.Title };

            await _dbContext.SaveAsync(news);

            return new APIGatewayHttpApiV2ProxyResponse
            {
                Body = System.Text.Json.JsonSerializer.Serialize(new RegisterNewsResponse(news.Id)),
                Headers = new Dictionary<string, string>
                {
                    {"Content-Type", "application/json"}
                },
                StatusCode = 200
            };
        }
        catch (Exception e)
        {
            return new APIGatewayHttpApiV2ProxyResponse
            {
                Body = @$"{{""message"": ""{e.Message}""}}",
                Headers = new Dictionary<string, string>
                    {
                        {"Content-Type", "application/json"},
                    },
                StatusCode = 400
            };
        }
    }
```

The developer deals with a lot of boilerplate code that usually is hidden from us. A similar code but using ASP.NET Core could be:

```csharp
        [HttpPost()]
        public async Task<RegisterNewsResponse> RegisterNews(RegisterNewsRequest request)
        {
            var news = new News() { Id = Guid.NewGuid(), Description = request.Description, Title = request.Title };

            await _dbContext.SaveAsync(news);

            return new RegisterNewsResponse(news.Id);
        }
```

As an option, we can decide to use ASP.NET Core to develop our Lambda functions, but there are drawbacks around it:

* There is an increment in the cold start time.
    
* There are features not supported by Lambda functions (SignalR, Blazor, etc).
    
* API Gateway is replaced by the routing mechanism present in ASP.NET.
    

So, [Lambda Annotations](https://github.com/aws/aws-lambda-dotnet/tree/master/Libraries/src/Amazon.Lambda.Annotations) are an effort to improve the development experience in writing Lambda functions that is similar to what we have with ASP.NET Core.

> Lambda Annotations is a programming model for writing .NET Lambda functions. At a high level, the programming model allows idiomatic .NET coding patterns. [C# Source Generators](https://learn.microsoft.com/en-us/dotnet/csharp/roslyn-sdk/source-generators-overview) are used to bridge the gap between the Lambda programming model to the Lambda Annotations programming model without adding any performance penalty.

To start using Lambda Annotations, we have a few pre-requisites:

* Have an [**IAM User**](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html#id_users_create_console) with programmatic access.
    
* Install the Amazon Lambda Templates (`dotnet new -i Amazon.Lambda.Templates`)
    
* Install the Amazon Lambda Tools (`dotnet tool install -g` [`Amazon.Lambda.Tools`](http://Amazon.Lambda.Tools))
    
* Install [**AWS SAM CLI**](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html).
    

Run the following commands:

```bash
dotnet new serverless.Annotations -n JournalApi -o .
dotnet new sln -n aws-lambda-annotations
dotnet sln add --in-root src/JournalApi
dotnet add src/JournalApi package AWSSDK.DynamoDBv2
```

By default, Lambda Annotations use JSON for the AWS SAM configuration file. The following steps are oriented to change that to use YAML. Open the solution and add a file named `template.yml` with the following content:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
```

Open the `aws-lambda-tools-defaults.json` file and edit the `template` property. The new value should be `template.yml`. Delete the file `serverless.template`. Open the `Functions.cs` file and replace the content with:

```csharp
using Amazon.Lambda.Core;
using Amazon.Lambda.Annotations;
using Amazon.Lambda.Annotations.APIGateway;
using Amazon.DynamoDBv2.DataModel;

[assembly: LambdaSerializer(typeof(Amazon.Lambda.Serialization.SystemTextJson.DefaultLambdaJsonSerializer))]

namespace JournalApi;

public record RegisterNewsRequest(string Description, string Title);

public record RegisterNewsResponse(Guid Id);

[DynamoDBTable("newstable")]
public class News
{
    [DynamoDBHashKey("id")]
    public Guid Id { get; set; }
    [DynamoDBProperty("description")]
    public string? Description { get; set; }
    [DynamoDBProperty("title")]
    public string? Title { get; set; }
}


public class Functions
{
    private readonly DynamoDBContext _dbContext;
    public Functions(DynamoDBContext dbContext)
    {
        _dbContext = dbContext;
    }

    [LambdaFunction(MemorySize = 512)]
    [HttpApi(LambdaHttpMethod.Post, "/")]
    public async Task<RegisterNewsResponse> RegisterNews([FromBody]RegisterNewsRequest request)
    {
        var news = new News() { Id = Guid.NewGuid(), Description = request.Description, Title = request.Title };

        await _dbContext.SaveAsync(news);

        return new RegisterNewsResponse(news.Id);
    }

    [LambdaFunction(MemorySize = 512)]
    [HttpApi(LambdaHttpMethod.Get, "/")]
    public async Task<List<News>> ListNews()
    {
        return await _dbContext.ScanAsync<News>(default).GetRemainingAsync();
    }

    [LambdaFunction(MemorySize = 512)]
    [HttpApi(LambdaHttpMethod.Get, "/{id}")]
    public async Task<News> GetNews(string id)
    {
        return await _dbContext.LoadAsync<News>(Guid.Parse(id));
    }
}
```

Open the `Startup.cs` file and replace the content with:

```csharp
using Amazon.DynamoDBv2.DataModel;
using Amazon.DynamoDBv2;
using Microsoft.Extensions.DependencyInjection;

namespace JournalApi;

[Amazon.Lambda.Annotations.LambdaStartup]
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddSingleton(new DynamoDBContext(new AmazonDynamoDBClient()));
    }
}
```

Compile the project and watch the magic of code generation by writing all the boilerplate code required by our Lambda functions:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1674409999263/eadbc9f8-df94-4001-a8ba-22a1f63e2d33.png align="center")

We suggest reviewing those classes to understand what is doing Lambda Annotations by us. But not only is generating code, open the `template.yml` to see all the configurations needed to deploy our Lambda functions. We are going to customize the file to include the creation of the DynamoDB table and the permissions needed by the Lambda functions:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Resources:
  NewsTable:
    Type: AWS::Serverless::SimpleTable
    Properties:
      PrimaryKey:
        Name: id
        Type: String
      TableName: newstable
  JournalApiFunctionsRegisterNewsGenerated:
    Type: AWS::Serverless::Function
    Metadata:
      Tool: Amazon.Lambda.Annotations
      SyncedEvents:
        - RootPost
    Properties:
      Runtime: dotnet6
      CodeUri: .
      MemorySize: 512
      Timeout: 30
      Policies:
        - AWSLambdaBasicExecutionRole
        - DynamoDBCrudPolicy:
            TableName: newstable
      PackageType: Zip
      Handler: JournalApi::JournalApi.Functions_RegisterNews_Generated::RegisterNews
      Events:
        RootPost:
          Type: HttpApi
          Properties:
            Path: /
            Method: POST
  JournalApiFunctionsListNewsGenerated:
    Type: AWS::Serverless::Function
    Metadata:
      Tool: Amazon.Lambda.Annotations
      SyncedEvents:
        - RootGet
    Properties:
      Runtime: dotnet6
      CodeUri: .
      MemorySize: 512
      Timeout: 30
      Policies:
        - AWSLambdaBasicExecutionRole
        - DynamoDBCrudPolicy:
            TableName: newstable
      PackageType: Zip
      Handler: JournalApi::JournalApi.Functions_ListNews_Generated::ListNews
      Events:
        RootGet:
          Type: HttpApi
          Properties:
            Path: /
            Method: GET
  JournalApiFunctionsGetNewsGenerated:
    Type: AWS::Serverless::Function
    Metadata:
      Tool: Amazon.Lambda.Annotations
      SyncedEvents:
        - RootGet
    Properties:
      Runtime: dotnet6
      CodeUri: .
      MemorySize: 512
      Timeout: 30
      Policies:
        - AWSLambdaBasicExecutionRole
        - DynamoDBCrudPolicy:
            TableName: newstable
      PackageType: Zip
      Handler: JournalApi::JournalApi.Functions_GetNews_Generated::GetNews
      Events:
        RootGet:
          Type: HttpApi
          Properties:
            Path: /{id}
            Method: GET
```

Go to the `template.yml` level and run the following commands to deploy it to AWS:

```bash
sam build
sam deploy --guided
```

And that's it. Our Lambda functions are up and running using Lambda Annotation. Sadly Lambda Annotations are still on preview, but we see a lot of potential for this initiative that wants to make easier the development experience for the .NET community. All the code is available [**here**](https://github.com/raulnq/aws-lambda-annotations)**.** Thanks, and happy coding.