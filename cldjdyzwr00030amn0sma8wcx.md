# GraphQL in .NET: Avoiding the n+1 query problem with data loaders

The N+1 query problem is a common performance pitfall when retrieving data from a database, which usually happens with an entity has associations (one to many or many to many). The complete case is like this: we run one query to get a list of objects and then run another query for each one to get the associated objects. Let's see how we can run into it with GraphQL. We will use the solution [here](https://github.com/raulnq/graphql-sandbox/tree/error-handling), download and open it.

Add two new models:

```csharp
public class Author
{
    public Guid Id { get; set; }
    public string Name { get; set; }
}

public class Comment
{
    public Guid Id { get; set; }
    public string Description { get; set; }
    public Guid PostId { get; set; }
}
```

Two new GraphQL types:

```csharp
public class AuthorType : ObjectType<Author>
{
    protected override void Configure(IObjectTypeDescriptor<Author> descriptor)
    {
        descriptor
            .Field(f => f.Name)
            .Type<StringType>();

        descriptor
            .Field(f => f.Id)
            .Type<IdType>();
    }
}

public class CommentType : ObjectType<Comment>
{
    protected override void Configure(IObjectTypeDescriptor<Comment> descriptor)
    {
        descriptor
            .Field(f => f.Description)
            .Type<StringType>();

        descriptor
            .Field(f => f.PostId)
            .Type<IdType>();

        descriptor
            .Field(f => f.Id)
            .Type<IdType>();
    }
}
```

And two new resolvers:

```csharp
public class AuthorQueries
{
    public Author? GetAuthor([Parent] Post post, [Service] Storage storage)
    {
        Console.WriteLine($"get author of post {post.Id}");

        return storage.Authors.FirstOrDefault(author => author.Id == post.AuthorId);
    }
}

public class CommentsQueries
{
    public List<Comment> GetComments([Parent] Post post, [Service] Storage storage)
    {
        Console.WriteLine($"get comments of post {post.Id}");

        return storage.Comments.Where(comments => comments.PostId == post.Id).ToList();
    }
}
```

Modify the `Post.cs` file with the following content:

```csharp
public class Post
{
    public Guid Id { get; set; }
    public string Title { get; set; }
    public string Body { get; set; }
    public List<Comment> Comments { get; set; }
    public Author Author { get; set; }
    public Guid AuthorId { get; set; }
}
```

And its corresponding GraphQL type:

```csharp
public class PostType : ObjectType<Post>
{
    protected override void Configure(IObjectTypeDescriptor<Post> descriptor)
    {
        descriptor
            .Field(f => f.Title)
            .Type<StringType>();

        descriptor
            .Field(f => f.Body)
            .Type<StringType>();

        descriptor
            .Field(f => f.Id)
            .Type<IdType>();

        descriptor
            .Field(f => f.Comments)
            .ResolveWith<CommentsQueries>(r => r.GetComments(default!, default!))
            .Type<ListType<CommentType>>();

        descriptor
            .Field(f => f.Author)
            .ResolveWith<AuthorQueries>(r => r.GetAuthor(default!, default!))
            .Type<AuthorType>();

        descriptor
            .Field(f => f.AuthorId)
            .Ignore();
    }
}
```

Add a couple of properties to our in-memory storage to support the new models:

```csharp
public class Storage
{
    public List<Post> Posts { get; set; } = new List<Post>();

    public List<Comment> Comments { get; set; } = new List<Comment>();

    public List<Author> Authors { get; set; } = new List<Author>();
}
```

And finally, update the `program.cs` file as follows:

```csharp
using api;

var builder = WebApplication.CreateBuilder(args);

var author = new Author() { Id= Guid.NewGuid(), Name="Name" };

var posts = Enumerable.Range(0, 20).Select(index => new Post() {
    Body = $"Body {index}", Id = Guid.NewGuid(), Title = $"Title {index}", AuthorId = author.Id
}).ToList();

var comments = new List<Comment>();

foreach (var post in posts)
{
    var list = Enumerable.Range(0, 5).Select(index => new Comment()
    {
        PostId = post.Id,
        Id = Guid.NewGuid(),
        Description = $"Description {post.Id} {index}"
    }).ToList();

    comments.AddRange(list);
}

builder.Services.AddSingleton(new Storage() { Posts = new List<Post>(posts), Comments = comments, Authors = new List<Author>() { author } });

builder.Services
    .AddGraphQLServer()
    .AddMutationConventions(applyToAllMutations: false)
    .AddQueryType<PostQueriesType>()
    .AddMutationType<PostMutationsType>()
    .AddFiltering()
    .AddSorting();

var app = builder.Build();

app.MapGraphQL();

app.Run();
```

Startup the application and run the following query:

```graphql
{
 posts {
   nodes{
      id,
      author {
        name
      }, comments {
        description
      }
   }
 } 
}
```

In the console window, you will see an output like this:

```bash
get all posts
get author of post 77959e95-9223-4ddd-806f-917077d87a84
get comments of post 77959e95-9223-4ddd-806f-917077d87a84
get author of post 2d1bb0fc-6d29-44c0-a820-fb59de92df2a
get comments of post 2d1bb0fc-6d29-44c0-a820-fb59de92df2a
get author of post ae9b5f2a-76ba-43c0-9f15-1b8b5b7f08ae
get comments of post ae9b5f2a-76ba-43c0-9f15-1b8b5b7f08ae
get author of post 70fab0f3-400b-42bc-bb7d-67909b356ad4
get comments of post 70fab0f3-400b-42bc-bb7d-67909b356ad4
get author of post d7ebcfc3-9a0a-4fcb-bfbf-21e4adefa68f
get comments of post d7ebcfc3-9a0a-4fcb-bfbf-21e4adefa68f
```

The n+1 query problem is in front of our eyes. Luckily Hot Chocolate implemented the solution for the problem, **data loaders**:

> With data loaders, we can now centralize the data fetching and reduce the number of round trips to our data source.
> 
> Instead of fetching the data from the repository directly, we fetch the data from the data loader. The data loader batches all the requests together into one request to the database.

There are two types of data loaders:

* **Batch data loader**: Used for one-to-one associations.
    
* **Group data loader**: User for one to many associations.
    

For `Authors`, we will use a batch data loader like this:

```csharp
public class AuthorBatchDataLoader : BatchDataLoader<Guid, Author>
{
    private readonly Storage _storage;

    public AuthorBatchDataLoader(
        Storage storage,
        IBatchScheduler batchScheduler,
        DataLoaderOptions? options = null)
        : base(batchScheduler, options)
    {
        _storage = storage;
    }

    protected override Task<IReadOnlyDictionary<Guid, Author>> LoadBatchAsync(
        IReadOnlyList<Guid> keys,
        CancellationToken cancellationToken)
    {
        Console.WriteLine($"get authors of posts {string.Join(",", keys.ToArray())}");

        var authors = _storage.Authors.Where(post => keys.Contains(post.Id));

        return Task.FromResult<IReadOnlyDictionary<Guid, Author>>(authors.ToDictionary(x => x.Id));
    }
}
```

Update the resolver to use the data loader:

```csharp
public class AuthorQueries
{
    public async Task<Author> GetAuthor([Parent] Post post, AuthorBatchDataLoader loader)
    {
        return await loader.LoadAsync(post.Id);
    }
}
```

For the `Comments`, we will use a group data loader like this:

```csharp
public class CommentGroupedDataLoader
    : GroupedDataLoader<Guid, Comment>
{
    private readonly Storage _repository;

    public CommentGroupedDataLoader(
        Storage repository,
        IBatchScheduler batchScheduler,
        DataLoaderOptions? options = null)
        : base(batchScheduler, options)
    {
        _repository = repository;
    }


    protected override Task<ILookup<Guid, Comment>> LoadGroupedBatchAsync(
        IReadOnlyList<Guid> keys,
        CancellationToken cancellationToken)
    {
        Console.WriteLine($"get comments of post {string.Join(",", keys.ToArray())}");

        var persons = _repository.Comments.Where(comment=> keys.Contains(comment.PostId));

        return Task.FromResult<ILookup<Guid,Comment>>(persons.ToLookup(x => x.PostId));
    }
}
```

And the resolver will change as follow:

```csharp
public class CommentsQueries
{
    public async Task<List<Comment>> GetComments([Parent] Post post, CommentGroupedDataLoader loader)
    {
        return (await loader.LoadAsync(post.Id)).ToList();
    }
}
```

Startup the application and run again the previous query to see the following outputs

```bash
get all posts
get comments of posts d6327235-1f5f-43e6-9e4a-4688e2334755,00d3ad13-97b6-4342-95af-246c9191fd96,b68435e9-8d64-4352-b2f7-57a9a9e47192,fb7eeac9-de79-4721-a540-47ceb231a9a5,84b8f72c-2e1d-4d42-a934-e1c6d7505ac2
get authors of posts d6327235-1f5f-43e6-9e4a-4688e2334755,00d3ad13-97b6-4342-95af-246c9191fd96,b68435e9-8d64-4352-b2f7-57a9a9e47192,fb7eeac9-de79-4721-a540-47ceb231a9a5,84b8f72c-2e1d-4d42-a934-e1c6d7505ac2
```

This time there is only three access to the storage. You can find more information about data loaders in the [official documentation](https://chillicream.com/docs/hotchocolate/v12/fetching-data/dataloader/). The resulting code is available [here](https://github.com/raulnq/graphql-sandbox/tree/dataloaders). Thanks, and happy coding.