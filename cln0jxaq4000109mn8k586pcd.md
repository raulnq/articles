---
title: "AWS Lambda and .NET: How to Structure Your Solution?"
datePublished: Tue Sep 26 2023 16:48:25 GMT+0000 (Coordinated Universal Time)
cuid: cln0jxaq4000109mn8k586pcd
slug: aws-lambda-and-net-how-to-structure-your-solution
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1695653309632/0cb411a6-23ed-4862-80c2-9266925cedd1.png
tags: net, aws-lambda, clean-architecture, localstack, nuke

---

As beginners writing Lambda functions using .NET, we often face questions about solution structure and the choice of libraries or tools. In this article, we address these concerns and present two solutions for the same problem - one following clean architecture principles and the other emphasizing a pragmatic approach.

## The Problem

We want to build an e-commerce application with the following requirements:

* A client can submit requests to register for the application.
    
* An administrator can approve the client's request.
    
* An administrator can register products and enable them.
    
* A client can add products to their shopping cart.
    
* A client can place an order.
    

## Tools

We are using a set of tools to improve the development process of our application:

* [Docker Desktop](https://www.docker.com/products/docker-desktop/)
    
* [AWS Serverless Application Model (SAM](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-getting-started.html)), find more information [here](https://blog.raulnq.com/deploying-aws-lambda-functions-with-aws-sam).
    
* [Local Stack](https://docs.localstack.cloud/getting-started/), find more information [here](https://blog.raulnq.com/running-aws-lambda-functions-locally-using-localstack).
    
* [Nuke](https://nuke.build/), take a look at our series [here](https://blog.raulnq.com/series/nuke).
    

## Libraries

In addition to the common `Amazon.Lambda.*` packages and the essential `Microsoft.Extensions.*` packages, we will use:

* [Entity Framework Core](https://learn.microsoft.com/en-us/ef/core/get-started/overview/first-app?tabs=netcore-cli)
    
* [Fluent Validation](https://docs.fluentvalidation.net/en/latest/)
    
* [SqlKata](https://sqlkata.com/docs)
    
* [NewId](https://masstransit.io/documentation/patterns/newid)
    
* [Amazon.Lambda.Annotations](https://github.com/aws/aws-lambda-dotnet/tree/master/Libraries/src/Amazon.Lambda.Annotations), find more information [here](https://blog.raulnq.com/reducing-boilerplate-code-with-lambda-annotations).
    
* [AWS.Lambda.Powertools](https://docs.powertools.aws.dev/lambda/dotnet/), find more information [here](https://blog.raulnq.com/enhance-aws-lambda-functions-using-powertools-for-net).
    
* [AWSSDK.Extensions.NETCore.Setup](https://docs.aws.amazon.com/sdk-for-net/v3/developer-guide/net-dg-config-netcore.html)
    
* [DbUp](https://dbup.readthedocs.io/en/latest/)
    

Disclaimer: Powertools for AWS Lambda and Lambda Annotations are not entirely compatible; we are only using the [logging](https://docs.powertools.aws.dev/lambda/dotnet/core/logging/) feature.

## The Solution

Both solutions consist of four projects:

* `_build`: This is the project where all automated tasks reside.
    
* `MyECommerceApp`: The main project. The structure will change based on the approach.
    
* `MyECommerceApp.Migrator`: A project responsible for keeping the database up-to-date.
    
* `MyECommerceApp.Tests`: This project contains all the acceptance tests.
    

The main driver in both approaches is building the application around **features**.

%[https://www.youtube.com/watch?v=5kOzZz2vj2o&ab_channel=NDCConferences] 

%[https://www.youtube.com/watch?v=L2Wnq0ChAIA&ab_channel=CodeOpinion] 

Another common aspect of both approaches is the **Test** project, which has the following structure:

```markdown
├── ClientRequests
    ├── ApproveClientRequestTests.cs
    ├── ClientRequestsDsl.cs
    └── RegisterClientRequestTests.cs
├── Clients
├── Infrastructure
├── Orders
├── Products
├── ShoppingCart
└── AppDsl.cs
```

Every folder contains tests per each **command/query** associated with a feature and the Domain Specific Language (DSL) of it. We can run the test with the command `Nuke Test`. Learn more about Acceptance Testing [here](https://blog.raulnq.com/understanding-acceptance-testing).

### Clean Architecture

There are numerous pieces of literature written about clean architecture, often referred to as onion or hexagonal architecture, which share the same advantages and disadvantages.

Pros:

* Loose coupling.
    
* The separation of concerns.
    
* The domain logic is agnostic and independent.
    

Cons:

* A high degree of [indirection](https://www.youtube.com/watch?v=DNjDZ0E6GUs&ab_channel=CodeOpinion).
    
* The learning curve, the pattern requires an upfront investment of time.
    
* In some cases, an over-engineered solution.
    
* Context switching, working on the same feature often requires navigating through multiple folders.
    

The first level of folders in the main project is reserved for the features.

```markdown
├── ClientRequests
├── Clients
├── Orders
├── Products
├── Shared
└── ShoppingCart
```

Within each feature folder, we can find the standard four layers of this architecture:

```markdown
├── ClientRequests
    ├── Application
    ├── Domain
    ├── Host
    └── Infrastructure
├── Clients
├── Orders
├── Products
├── Shared
└── ShoppingCart
```

* The **Domain Layer** contains the business logic of our application and should be independent of any other layers.
    
* The **Application Layer** implements the use cases and orchestrates the flow of data between the Domain Layer and the Infrastructure Layer.
    
* The **Infrastructure Layer** interacts with the outside world. This layer is responsible for tasks, such as database access, API calls, message handling, etc.
    
* The **Presentation Layer** is responsible for displaying data to users and collecting input from them, as well as from other sources.
    

```markdown
├── ClientRequests
    ├── Application  
        ├── ApproveClientRequest.cs
        └── RegisterClientRequest.cs
    ├── Domain
        ├── ClientRequest.cs
        ├── ClientRequestApproved.cs
        └── ClientRequestStatus.cs
    ├── Host
        └── Function.cs
    └── Infrastructure
        ├── AnyClientRequests.cs
        ├── EntityTypeConfiguration.cs
        ├── GetClientRequest.cs
        ├── ListClientRequests.cs
        └── ServiceCollectionExtensions.cs
├── Clients
├── Orders
├── Products
├── Shared
└── ShoppingCart
```

In the **Application** folder, all the **commands** are stored. Each file maintains a consistent structure, such as:

```csharp
public static class RegisterClientRequest
{
    public class Command
    {
        public string Name { get; set; }
        public string Address { get; set; }
        public string PhoneNumber { get; set; }
        [JsonIgnore]
        public bool Any { get; set; }
    }

    public class Validator : AbstractValidator<Command>
    {
        public Validator()
        {
            RuleFor(command => command.Name).MaximumLength(100).NotEmpty();
            RuleFor(command => command.Address).MaximumLength(500).NotEmpty();
            RuleFor(command => command.PhoneNumber).MaximumLength(20).NotEmpty();
        }
    }

    public class Result
    {
        public Guid ClientRequestId { get; set; }
    }

    public class Handler
    {
        private readonly IRepository<ClientRequest> _clientRequestRepository;

        public Handler(IRepository<ClientRequest> clientRequestRepository)
        {
            _clientRequestRepository = clientRequestRepository;
        }

        public Task<Result> Handle(Command command)
        {
            var clientRequest = new ClientRequest (
                NewId.Next().ToSequentialGuid(), 
                command.Name, 
                command.Address, 
                command.PhoneNumber, 
                command.Any);

            _clientRequestRepository.Add(clientRequest);

            return Task.FromResult(new Result()
            {
                ClientRequestId = clientRequest.ClientRequestId
            });
        }
    }
}
```

In the **Infrastructure** folder, everything related to a specific technology is stored: the queries, the Entity Framework configuration, and the dependency injection setup. The **queries** follow a structure like this:

```csharp
public class GetClientRequest
{
    public class Query
    {
        public Guid ClientRequestId { get; set; }
    }

    public class Result
    {
        public Guid ClientRequestId { get; set; }
        public string Name { get; set; }
        public string Address { get; set; }
        public string PhoneNumber { get; set; }
        public string Status { get; set; }
        public DateTimeOffset RegisteredAt { get; set; }
        public DateTimeOffset? ApprovedAt { get; set; }
        public DateTimeOffset? RejectedAt { get; set; }
    }

    public class Runner : BaseRunner
    {
        public Runner(SqlKataQueryRunner queryRunner) : base(queryRunner) { }

        public Task<Result> Run(Query query)
        {
            return _queryRunner.Get<Result>((qf) => qf
                .Query(Tables.ClientRequests)
                .Where(Tables.ClientRequests.Field(nameof(Query.ClientRequestId)), query.ClientRequestId));
        }
    }
}
```

The entry points for our Lambda functions are in the **Host** folder, a single file like this:

```csharp
public class Function : BaseFunction
{
    [LambdaFunction]
    [RestApi(LambdaHttpMethod.Post, "/client-requests")]
    public Task<IHttpResult> RegisterClientRequest(
        [FromServices] AnyClientRequests.Runner runner,
        [FromServices] TransactionBehavior behavior, 
        [FromServices] RegisterClientRequest.Handler handler,
        [FromBody] RegisterClientRequest.Command command)
    {
        return Handle(async ()=>
        {
            new RegisterClientRequest.Validator().ValidateAndThrow(command);
            command.Any = await runner.Run(new AnyClientRequests.Query() { Name = command.Name });
            var result = await behavior.Handle(() => handler.Handle(command));
            return result;
        });
    }

    [LambdaFunction]
    [RestApi(LambdaHttpMethod.Post, "/client-requests/{clientRequestId}/approve")]
    public Task<IHttpResult> ApproveClientRequest(
    [FromServices] TransactionBehavior behavior,
    [FromServices] ApproveClientRequest.Handler handler,
    [FromServices] EventPublisher publisher,
    string clientRequestId)
    {
        return Handle(async () =>
        {
            var command = new ApproveClientRequest.Command() { ClientRequestId = Guid.Parse(clientRequestId) };
            await behavior.Handle(() => handler.Handle(command));
            await publisher.Publish(new ClientRequestApproved(command.ClientRequestId));
        });
    }

    [LambdaFunction]
    [RestApi(LambdaHttpMethod.Get, "/client-requests")]
    public Task<IHttpResult> ListClientRequests(
        [FromServices] ListClientRequests.Runner runner, 
        [FromQuery] string name,
        [FromQuery] int page, 
        [FromQuery] int pageSize)
    {
        return Handle(()=>runner.Run(new ListClientRequests.Query() { Name = name, Page = page, PageSize = pageSize }));
    }
}
```

Remember, we are using Lambda Annotations, so the library generates all the boilerplate code for us. The **Domain** folder contains the classes related to the core of the application. Finally, there is a **Shared** folder containing concerns from every layer that can be shared among various components:

```markdown
├── Shared
    ├── Domain
        ├── Exceptions.cs
        └── IRepository.cs
    ├── Host
        └── BaseFunction.cs
    └── Infrastructure
        ├── EntityFramework
        ├── ExceptionHandling
        ├── Messaging
        ├── SqlKata
        └── StringExtensions.cs
```

The code for this solution can be found [here](https://github.com/raulnq/aws-clean-arch-lambda).

### Pragmatic

The second approach involves removing the concept of layers from the solution to keep all the classes directly under the same feature folder:

```markdown
├── ClientRequests
    ├── AnyClientRequests.cs
    ├── ApproveClientRequest.cs
    ├── ClientRequest.cs
    ├── EntityTypeConfiguration.cs
    ├── GetClientRequest.cs
    ├── ListClientRequests.cs
    ├── RegisterClientRequest.cs
    └── ServiceCollectionExtensions.cs
├── Clients
├── Infrastructure
├── Orders
├── Products
└── ShoppingCart
```

Now, the **commands** and **queries** can incorporate the Lambda function as part of their respective files:

```csharp
public class RegisterClientRequest : BaseFunction
{
    public class Command
    {
        public string Name { get; set; }
        public string Address { get; set; }
        public string PhoneNumber { get; set; }
        [JsonIgnore]
        public bool Any { get; set; }
    }

    public class Validator : AbstractValidator<Command>
    {
        public Validator()
        {
            RuleFor(command => command.Name).MaximumLength(100).NotEmpty();
            RuleFor(command => command.Address).MaximumLength(500).NotEmpty();
            RuleFor(command => command.PhoneNumber).MaximumLength(20).NotEmpty();
        }
    }

    public class Result
    {
        public Guid ClientRequestId { get; set; }
    }

    public class Handler
    {
        private readonly ApplicationDbContext _context;

        public Handler(ApplicationDbContext context)
        {
            _context = context;
        }

        public Task<Result> Handle(Command command)
        {
            var clientRequest = new ClientRequest(
                NewId.Next().ToSequentialGuid(),
                command.Name,
                command.Address,
                command.PhoneNumber,
                command.Any);

            _context.Set<ClientRequest>().Add(clientRequest);

            return Task.FromResult(new Result()
            {
                ClientRequestId = clientRequest.ClientRequestId
            });
        }
    }

    [LambdaFunction]
    [RestApi(LambdaHttpMethod.Post, "/client-requests")]
    public Task<IHttpResult> Handle(
        [FromServices] AnyClientRequests.Runner runner,
        [FromServices] TransactionBehavior behavior,
        [FromServices] Handler handler,
        [FromBody] Command command)
    {
        return Handle(async () =>
        {
            new Validator().ValidateAndThrow(command);
            command.Any = await runner.Run(new AnyClientRequests.Query() { Name = command.Name });
            var result = await behavior.Handle(() => handler.Handle(command));
            return result;
        });
    }
}
```

```csharp
public class ListClientRequests : BaseFunction
{
    public class Query : ListQuery
    {
        public string Name { get; set; }
    }

    public class Result
    {
        public Guid ClientRequestId { get; set; }
        public string Name { get; set; }
        public string Address { get; set; }
        public string PhoneNumber { get; set; }
        public string Status { get; set; }
        public DateTimeOffset RegisteredAt { get; set; }
        public DateTimeOffset? ApprovedAt { get; set; }
        public DateTimeOffset? RejectedAt { get; set; }
    }

    public class Runner : BaseRunner
    {
        public Runner(SqlKataQueryRunner queryRunner): base(queryRunner) { }

        public Task<ListResults<Result>> Run(Query query)
        {
            return _queryRunner.List<Query, Result>((qf)=> {
                var statement = qf.Query(Tables.ClientRequests);

                if (!string.IsNullOrEmpty(query.Name))
                {
                    statement = statement.WhereLike(Tables.ClientRequests.Field(nameof(Query.Name)), $"%{query.Name}%");
                }
                return statement;
            }, query);
        }
    }

    [LambdaFunction]
    [RestApi(LambdaHttpMethod.Get, "/client-requests")]
    public Task<IHttpResult> Handle(
        [FromServices] Runner runner,
        [FromQuery] string name,
        [FromQuery] int page,
        [FromQuery] int pageSize)
    {
        return Handle(() => runner.Run(new Query() { Name = name, Page = page, PageSize = pageSize }));
    }
}
```

The **Shared** folder on the first approach was replaced with an **Infrastructure** folder with:

```markdown
├── Infrastructure
    ├── EntityFramework
    ├── ExceptionHandling
    ├── Host
    ├── Messaging
    ├── SqlKata
    └── StringExtensions.cs
```

With this approach, **we keep all the elements that could potentially be impacted by a change in a feature together**. The code for this solution can be found [here](https://github.com/raulnq/aws-pragmatic-lambda).

In conclusion, structuring an AWS Lambda and .NET solution can be approached using either clean architecture or a more pragmatic method. Both have their advantages and drawbacks, with clean architecture promoting loose coupling and separation of concerns, while the pragmatic approach emphasizes simplicity and cohesiveness. Ultimately, the choice depends on your specific needs, development style, and project requirements. Thanks, and happy coding.