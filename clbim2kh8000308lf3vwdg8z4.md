# GraphQL in .NET: Error Handling

In our journey with GraphQL, sooner or later, we are going to need to pick a strategy to handle errors during mutations. There is a great post ["A Guide to GraphQL Errors" ](https://xuorig.medium.com/a-guide-to-graphql-errors-bb9ba9f15f85) where the author shows several options (its pros and cons). Based on this, we will implement the last stage presented there: **Stage 6a: Error Union List + Interface**. 

We will modify the code presented [here](https://github.com/raulnq/graphql-sandbox/tree/data), please clone or download it. Open the solution and add a file named `IError.cs` with the following content:

``` csharp
public interface IError
{
    string Message { get; set; }
}

public class NotEmptyError : IError
{
    public string Message { get; set; } = null!;

    public NotEmptyError(string field)
    {
        Message= $"The {field} is required";
    }
}

public class MaxLengthError : IError
{
    public string Message { get; set; } = null!;

    public MaxLengthError(string field, int maxLength)
    {
        Message = $"The max length for {field} is {maxLength}";
    }
}
``` 

Open the `PostMutations.cs` file, and modify the `AddPostPayload` record  as follow:

```csharp
public record AddPostPayload(Post? post, IError[]? errors);
``` 

In the same file, modify the `AddPost` method with the following content:

```csharp
public AddPostPayload AddPost(AddPostInput input, [Service] Storage storage)
{
    var errors = new List<IError>();

    if(string.IsNullOrEmpty(input.Body))
    {
        errors.Add(new NotEmptyError(nameof(AddPostInput.Body)));
    }

    if (string.IsNullOrEmpty(input.Title))
    {
        errors.Add(new NotEmptyError(nameof(AddPostInput.Title)));
    }

    if (input.Body?.Length >= 256)
    {
        errors.Add(new MaxLengthError(nameof(AddPostInput.Body), 256));
    }

    if (input.Title?.Length >= 1024)
    {
        errors.Add(new MaxLengthError(nameof(AddPostInput.Title), 1024));
    }

    if (errors.Any())
    {
        return new AddPostPayload(null, errors.ToArray());
    }

    var post = new Post() { Id = Guid.NewGuid(), Body = input.Body, Title = input.Title };

    storage.Posts.Add(post);

    return new AddPostPayload(post, null);
}
``` 

All the code to support the errors is done, time to setup the schema to support it. Start creating a file named  `ErrorType.cs`. Here we are going to register an [interface](https://chillicream.com/docs/hotchocolate/v12/defining-a-schema/interfaces), two errors and an [union](https://chillicream.com/docs/hotchocolate/v12/defining-a-schema/unions):

```csharp
public class ErrorType : InterfaceType<IError>
{
    protected override void Configure(
        IInterfaceTypeDescriptor<IError> descriptor)
    {
    }
}

public class MaxLengthErrorType : ObjectType<MaxLengthError>
{
    protected override void Configure(
        IObjectTypeDescriptor<MaxLengthError> descriptor)
    {
        descriptor.Implements<ErrorType>();
    }
}

public class NotEmptyErrorType : ObjectType<NotEmptyError>
{
    protected override void Configure(
        IObjectTypeDescriptor<NotEmptyError> descriptor)
    {
        descriptor.Implements<ErrorType>();
    }
}

public class AddPostPayloadErrorType : UnionType
{
    protected override void Configure(IUnionTypeDescriptor descriptor)
    {
        descriptor.Name("AddPostPayloadError");
        descriptor.Type<MaxLengthErrorType>();
        descriptor.Type<NotEmptyErrorType>();
    }
}
``` 

Create a file named `AddPostPayloadType.cs` with the following content:

```csharp
public class AddPostPayloadType : ObjectType<AddPostPayload>
{
    protected override void Configure(IObjectTypeDescriptor<AddPostPayload> descriptor)
    {
        descriptor
            .Field(f => f.post)
            .Type<PostType>();

        descriptor
            .Field(f=>f.errors)
            .Type<ListType<AddPostPayloadErrorType>>();
    }
}
``` 

And finally, update the `PostMutationsType.cs` file as follows:

```csharp
public class PostMutationsType : ObjectType<PostMutations>
{
    protected override void Configure(
        IObjectTypeDescriptor<PostMutations> descriptor)
    {
        descriptor.Field(f => f.AddPost(default!, default!)).Type<AddPostPayloadType>();
        descriptor.Field(f => f.RemovePost(default!, default!));
        descriptor.Field(f => f.EditPost(default!, default!));
    }
}
``` 

Run the solution, go to `https://localhost:7121/graphql/` and run the following mutation:

```
mutation {
  addPost(input: { body: "", title: "" }) {
    post {
      id
    }
    errors {
      ... on MaxLengthError {
        message
      }
      ... on NotEmptyError {
        message
      }
    }
  }
}
``` 

To see a result such as:

```json
{
  "data": {
    "addPost": {
      "post": null,
      "errors": [
        {
          "message": "The Body is required"
        },
        {
          "message": "The Title is required"
        }
      ]
    }
  }
}
``` 

We did it! but there is a lot of boilerplate code. Luckily, [HotChocolate](https://chillicream.com/docs/hotchocolate/v12) has an implementation of this pattern out of the box with a difference, the errors are going to be thrown as exceptions. Create a new file `NotFoundException.cs` as follows:

```csharp
public class NotFoundException : Exception
{
    public NotFoundException(Guid id) : base($"The with {id} was not found")
    {
    }
}
``` 

Open the `PostMutations.cs` file and modify the `EditPost` method with the following content:

```csharp
public Post EditPost(EditPostInput input, [Service] Storage storage)
{
    var post = storage.Posts.FirstOrDefault(post => post.Id == input.Id);
    if(post==null)
    {
        throw new NotFoundException(input.Id);
    }
    post.Body = input.Body;
    return post;
}
``` 

Go to the `PostMutationsType.cs` file and update the code as follow:

```csharp
public class PostMutationsType : ObjectType<PostMutations>
{
    protected override void Configure(
        IObjectTypeDescriptor<PostMutations> descriptor)
    {
        descriptor.Field(f => f.AddPost(default!, default!)).Type<AddPostPayloadType>();
        descriptor.Field(f => f.RemovePost(default!, default!));
        descriptor.Field(f => f.EditPost(default!, default!))
            .UseMutationConvention()
            .Error<NotFoundException>();
    }
}
```

Finally, go to the `Program.cs` file and enable the mutation conventions with `AddMutationConventions`:

```csharp
builder.Services
    .AddGraphQLServer()
    .AddMutationConventions(applyToAllMutations: false)
    .AddQueryType<PostQueriesType>()
    .AddMutationType<PostMutationsType>()
    .AddFiltering()
    .AddSorting();
``` 

Run the solution and run the following mutation:

```
mutation {
  editPost(input: { body: "abc", id: "5e629e6fb8d64126196c7d12cb6f8a6a" }) {
    post {
      id
    }
    errors {
      ... on NotFoundError {
        message
      }
    }
  }
}
``` 

To get the same result as our manual implementation:

```json
{
  "data": {
    "editPost": {
      "post": null,
      "errors": [
        {
          "message": "The with 5e629e6f-b8d6-4126-196c-7d12cb6f8a6a was not found"
        }
      ]
    }
  }
}
```

You can check more about this feature [here](https://chillicream.com/docs/hotchocolate/v12/defining-a-schema/mutations/#errors). The final code of this post is available [here](https://github.com/raulnq/graphql-sandbox/tree/error-handling). Thanks, and happy coding.

