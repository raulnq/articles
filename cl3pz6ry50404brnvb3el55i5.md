## AWS IAM Database Authentication with EF Core

[AWS Aurora](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/CHAP_AuroraOverview.html) (both MySQL and [Postgres](https://www.postgresql.org/)) and MariaDB allow [IAM authentication](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/UsingWithRDS.IAMDBAuth.html). With this authentication method, you don't need to use a password when you connect to a database. Instead, you use an authentication token. In this post, we are going to explore how to use IAM database authentication with [EF Core](https://docs.microsoft.com/en-us/ef/core/) and Aurora Postgres.

Before starting, you need to fulfill a few pre-requisites:

- Enabling [IAM Authentication](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/UsingWithRDS.IAMDBAuth.Enabling.html) in your Aurora Postgres database.
- Have a [IAM User](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html#id_users_create_console) with programmatic access.
- Setup your [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-creds) locally.

The first step is to create an IAM policy using this template:

```json
{
   "Version": "2012-10-17",
   "Statement": [
      {
         "Effect": "Allow",
         "Action": [
             "rds-db:connect"
         ],
         "Resource": [
             "arn:aws:rds-db:[region]:[account-id]:dbuser:[db-cluster-resourceId]/[db-user-name]"
         ]
      }
   ]
}
``` 

- `[region]` is the AWS Region for the database cluster. An example of a region is `us-east-2`.
- `[account-id]` is the AWS account number for the database cluster. An example of an account number is `1234567890`.
- `[db-cluster-resourceId]` is the identifier for the database cluster. An example of this identifier is `cluster-ABCDEFGHIJKL01234`.
- `[db-user-name]` is the name of the database account to associate with IAM authentication.

Once we have that, we need to attach it to our IAM user. The next step is to create a database (we are using the `postgres` user to run the following scripts):

```sql
CREATE DATABASE tasksmanager
WITH 
ENCODING = 'UTF8';
``` 

And then a table inside this new database:

```sql
CREATE TABLE tasks(
  id UUID PRIMARY KEY,
  name VARCHAR(255)
);
``` 

And finally, the database user:

```sql
CREATE USER db_iam_user; 
GRANT rds_iam TO db_iam_user;
GRANT postgres TO db_iam_user;
``` 

Let's do some code now,  we are going to create a .NET ASP Core Web API application. Add the following NuGet packages to that application:

- [AWSSDK.RDS](https://www.nuget.org/packages/AWSSDK.RDS/).
- [Npgsql.EntityFrameworkCore.PostgreSQL](https://www.nuget.org/packages/Npgsql.EntityFrameworkCore.PostgreSQL).
- [EFCore.NamingConventions](https://www.nuget.org/packages/EFCore.NamingConventions/).

In our `appsetting.json`, we are going to add a `ConnectionString` property:

``` json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*",
  "ConnectionString": "host=[aurora-database-endpoint];Database=tasksmanager;Username=db_iam_user;SSL Mode=Require;TrustServerCertificate=true"
}

```
Add the a file with the following content:

```csharp
    public class Task
    {
        public Guid Id { get; set; }
        public string Name { get; set; }

        private Task()
        {

        }

        public Task(string name)
        {
            Id = Guid.NewGuid();
            Name = name;
        }
    }
``` 

With that, we can create our `DbContext` class:

```csharp
    public class ApplicationDbContext : DbContext
    {
        public DbSet<Task> Tasks { get; set; }

        public ApplicationDbContext(DbContextOptions options) : base(options)
        {

        }
    }
``` 
Now, add a `ServiceCollectionExtensions.cs` file. Here is where all the magic happens:

```csharp
    public static class ServiceCollectionExtensions
    {
        public static IServiceCollection AddPostgreSQL(this IServiceCollection services, IConfiguration configuration)
        {
            services.AddDbContext<ApplicationDbContext>(options =>
            {
                var connectionString = configuration["ConnectionString"];
                options.UseNpgsql(connectionString, npgsqlOption => { npgsqlOption.UseAwsIamAuthentication(); });
                options.UseLowerCaseNamingConvention();
            });
            return services;
        }

        public static NpgsqlDbContextOptionsBuilder UseAwsIamAuthentication(this NpgsqlDbContextOptionsBuilder optionsBuilder)
        {
            optionsBuilder.ProvidePasswordCallback(RequestAwsIamAuthToken);
            return optionsBuilder;
        }

        static string RequestAwsIamAuthToken(string host, int port, string database, string username)
        {
            return RDSAuthTokenGenerator.GenerateAuthToken(host, port, username);
        }
    }
``` 

And then, add the following line to your ` Program.cs` file:

```csharp
builder.Services.AddPostgreSQL(builder.Configuration);
``` 

Add a couple of models for our request and response:

```csharp
    public class RegisterTaskCommand
    {
        public string Name { get; set; }
    }

    public class ListTasksResult
    {
        public Guid Id { get; set; }
        public string Name { get; set; }
    }
``` 

And finally, our `TaskController.cs` file:


```csharp
    [ApiController]
    [Route("[controller]")]
    public class TasksController : ControllerBase
    {
        private readonly ApplicationDbContext _context;
        public TasksController(ApplicationDbContext context)
        {
            _context = context;
        }

        [HttpGet()]
        public Task<ListTasksResult[]> ListTasks()
        {
            return _context.Tasks.Select(t => new ListTasksResult() { Id = t.Id, Name = t.Name }).ToArrayAsync();
        }

        [HttpPost()]
        public async Task<Guid> RegisterTask(RegisterTaskCommand command)
        {
            var task = new Task(command.Name);
            _context.Tasks.Add(task);
            await _context.SaveChangesAsync();
            return task.Id;
        }
    }
``` 

That's it, run your application and test your passwordless connection against your database. [Here](https://github.com/raulnq/aws-aurora-sandbox), you can find all the code and database scripts.


