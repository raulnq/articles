# Accessing a non-public Amazon Aurora database in a VPC from a Lambda function

The scenarios where we use AWS Lambda functions are growing every day. By default, AWS does not launch Lambda functions within a Virtual Private Cloud (VPC), so they can only connect to public resources accessible through the internet. But for security reasons, many resources are kept inaccessible from the internet and only accessible from within a VPC. Examples of those resources are databases, cache instances, or internal services.

Luckily, AWS Lambda functions allow us to change the default behavior and configure them within a VPC. But there is a catch the Lambda function will lose access to the internet (and many services are accessible from the internet, such as DynamoDB, SNS, SQS, etc.) There are ways to recover access to the internet or specific AWS services, but in general, unless needed, do not set up Lambda function in VPC.

## Prerequisites

* An Amazon Aurora Server (PostgreSQL compatible edition with [**IAM Authentication**](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/UsingWithRDS.IAMDBAuth.Enabling.html) enabled)**.**
    
* An [**IAM User**](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html#id_users_create_console) with programmatic access.
    
* Install [AWS **CLI**](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-creds).
    
* Install the Amazon Lambda Templates (`dotnet new -i Amazon.Lambda.Templates`).
    
* Install the Amazon Lambda Tools (`dotnet tool install -g Amazon.Lambda.Tools`).
    
* Install [**AWS SAM CLI**](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html).
    

## Database setup

Create a database (we are using the `postgres` user to run the following scripts):

```sql
CREATE DATABASE journal
WITH 
ENCODING = 'UTF8';
```

Then a table inside the new database:

```sql
CREATE TABLE posts(
  id UUID PRIMARY KEY,
  title VARCHAR(255),
  description VARCHAR(1024)
);
```

And finally, the database user:

```sql
CREATE USER db_iam_user; 
GRANT rds_iam TO db_iam_user;
GRANT postgres TO db_iam_user;
```

## Let's code

We will create two AWS Lambda Functions, one to register a post and the other to list them. We will store the data in the database previously created. And to avoid using a password during the connection against the database, we will use [IAM Authentication](https://blog.raulnq.com/aws-iam-database-authentication-with-ef-core). We will use [AWS SAM](https://blog.raulnq.com/deploying-aws-lambda-functions-with-aws-sam) as a deployment mechanism. Run the following command to create the .NET projects and solution:

```bash
dotnet new lambda.EmptyFunction -n RegisterPost -o .
dotnet new lambda.EmptyFunction -n ListPosts -o .
dotnet new classlib -n Library -o src/Library
dotnet new sln -n aurora-sandbox
dotnet sln add --in-root src/RegisterPost 
dotnet sln add --in-root src/ListPosts
dotnet sln add --in-root src/Library
```

Add the following NuGet packages to each project:

```bash
dotnet add src/Library package AWSSDK.RDS
dotnet add src/Library package Dapper
dotnet add src/Library package Npgsql
dotnet add src/RegisterPost package Amazon.Lambda.APIGatewayEvents
dotnet add src/ListPosts package Amazon.Lambda.APIGatewayEvents
```

And the references between projects:

```bash
dotnet add src/RegisterPost/RegisterPost.csproj reference src/Library/Library.csproj
dotnet add src/ListPosts/ListPosts.csproj reference src/Library/Library.csproj
```

Open the solution and under the `Library` project, add a `Post.cs` file with the following content:

```csharp
public class Post
{
    public Guid Id { get; set; }
    public string? Title { get; set; }
    public string? Description { get; set; }
}
```

In the same project add a `PostsRepository.cs` file as follows:

```csharp
public class PostsRepository
{
    private readonly NpgsqlConnection _connection;

    public PostsRepository(string connectionString)
    {
        _connection = new NpgsqlConnection(connectionString);

        _connection.ProvidePasswordCallback = RequestAwsIamAuthToken;
    }

    private string RequestAwsIamAuthToken(string host, int port, string database, string username)
    {
        return RDSAuthTokenGenerator.GenerateAuthToken(host, port, username);
    }

    public Task Create(Post post)
    {
        return _connection.ExecuteAsync("INSERT INTO public.posts(id, title, description) VALUES (@Id, @Title, @Description)", post);
    }

    public Task<IEnumerable<Post>> List()
    {
        return _connection.QueryAsync<Post>("SELECT id, title, description from public.posts");
    }
}
```

Under the `RegisterPost` project, modify the `Function.cs` file as follow:

```csharp
public class Function
{
    private PostsRepository _repository;

    public Function()
    {
        var host = Environment.GetEnvironmentVariable("DB_HOST");
        var database = Environment.GetEnvironmentVariable("DB_NAME");
        var user = Environment.GetEnvironmentVariable("DB_USER");
        _repository = new PostsRepository($"host={host};Port=5432;Database={database};Username={user};SSL Mode=Require;TrustServerCertificate=true");
    }

    public async Task<APIGatewayHttpApiV2ProxyResponse> FunctionHandler(APIGatewayHttpApiV2ProxyRequest input, ILambdaContext context)
    {
        var request = JsonSerializer.Deserialize<RegisterPostRequest>(input.Body)!;

        var post = new Post() { Id = Guid.NewGuid(), Description = request.Description, Title = request.Title };

        await _repository.Create(post);

        var response = JsonSerializer.Serialize(new RegisterPostResponse(post.Id));

        return new APIGatewayHttpApiV2ProxyResponse
        {
            Body = response,
            StatusCode = 200,
            Headers = new Dictionary<string, string> { { "Content-Type", "application/json" } }
        };
    }

    public record RegisterPostRequest(string Description, string Title);

    public record RegisterPostResponse(Guid Id);
}
```

Under the `LisPosts` project, modify the `Function.cs` file as follow:

```csharp
public class Function
{
    private PostsRepository _repository;

    public Function()
    {
        var host = Environment.GetEnvironmentVariable("DB_HOST");
        var database = Environment.GetEnvironmentVariable("DB_NAME");
        var user = Environment.GetEnvironmentVariable("DB_USER");
        _repository = new PostsRepository($"host={host};Port=5432;Database={database};Username={user};SSL Mode=Require;TrustServerCertificate=true");
    }

    public async Task<APIGatewayHttpApiV2ProxyResponse> FunctionHandler(APIGatewayHttpApiV2ProxyRequest input, ILambdaContext context)
    {
        var posts = await _repository.List();

        var response = JsonSerializer.Serialize(posts);

        return new APIGatewayHttpApiV2ProxyResponse
        {
            Body = response,
            StatusCode = 200,
            Headers = new Dictionary<string, string> { { "Content-Type", "application/json" } }
        };
    }
}
```

And finally, create a `template.yml` file at the solution level:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: >
  Posts API

Parameters:
  User:
    Type: String
    Default: db_iam_user
    Description: Database user
  Name:
    Type: String
    Default: journal
    Description: Database name
  Host:
    Type: String
    Default: <DATABASE_HOST>
    Description: Database host prefix
  Cluster:
    Type: String
    Default: <DATABASE_RESOURCEID>
    Description: Database cluster
  Vpc:
    Type: String
    Default: <VPCID>
    Description: RDS Vpc
  Subnet:
    Type: String
    Default: <SUBNETID>
    Description: RDS Subnet

Globals:
  Function:
    Timeout: 60
    MemorySize: 512
    Runtime: dotnet6
    Architectures:
      - x86_64
    Environment:
      Variables:
        DB_USER: !Ref User
        DB_HOST: !Sub "${Host}.${AWS::Region}.rds.amazonaws.com"
        DB_NAME: !Ref Name

Resources:
  RDSPasswordlessPolicy:
    Type: 'AWS::IAM::ManagedPolicy'
    Properties:
      PolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Action:
              - rds-db:connect
            Resource: !Sub "arn:aws:rds-db:${AWS::Region}:${AWS::AccountId}:dbuser:${Cluster}/${User}"

  LambdaSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: lambda security group
      VpcId: !Ref Vpc

  RegisterPostFunction:
    Type: AWS::Serverless::Function
    Properties:
      Runtime: dotnet6
      Handler: RegisterPost::RegisterPost.Function::FunctionHandler
      CodeUri: ./src/RegisterPost/
      VpcConfig:
        SecurityGroupIds:
          - !Ref LambdaSecurityGroup
        SubnetIds:
          - !Ref Subnet
      Policies:
        - AWSLambda_FullAccess
        - AWSLambdaVPCAccessExecutionRole
        - !Ref RDSPasswordlessPolicy
      Events:
        RegisterPost:
          Type: Api
          Properties:
            Path: /posts
            Method: post

  ListPostsFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: ListPosts::ListPosts.Function::FunctionHandler
      CodeUri: ./src/ListPosts/
      VpcConfig:
        SecurityGroupIds:
          - !Ref LambdaSecurityGroup
        SubnetIds:
          - !Ref Subnet
      Policies:
        - AWSLambda_FullAccess
        - AWSLambdaVPCAccessExecutionRole
        - !Ref RDSPasswordlessPolicy
      Events:
        ListPosts:
          Type: Api
          Properties:
            Path: /posts
            Method: get

Outputs:
  PostsApi:
    Description: "API Gateway endpoint URL for Prod stage for posts api"
    Value: !Sub "https://${ServerlessRestApi}.execute-api.${AWS::Region}.amazonaws.com/Prod/posts/"
```

To connect the Lambda function to the VPC, we did three things:

* Assign the `AWSLambdaVPCAccessExecutionRole` and `AWSLambda_FullAccess` policies.
    
* Create the `LambdaSecurityGroup` security group(inside our VPC) and use it in our Lambda function(`SecurityGroupIds` property).
    
* Assign a subnet(the same subnet where our database lives) to the Lambda function(`SubnetIds` property)
    

To use the IAM Authentication, we created the `RDSPasswordlessPolicy` policy and assigned it to the Lambda function. At the solution level, run `sam build` to build the serverless application, and then `sam deploy --guided`, the command will guide you through the deployment. And that's it; we should have a Lambda function with access to a VPC resource. All the code is available [here](https://github.com/raulnq/aws-lambda-aurora). Thanks, and happy coding.