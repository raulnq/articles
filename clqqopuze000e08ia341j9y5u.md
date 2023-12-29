---
title: ".NET 8: Working with HTMX and Razor Components"
datePublished: Fri Dec 29 2023 13:43:44 GMT+0000 (Coordinated Universal Time)
cuid: clqqopuze000e08ia341j9y5u
slug: net-8-working-with-htmx-and-razor-components
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1703599610345/f858b1fe-5ae5-4e93-af01-daafa99f0b43.png
tags: net, htmx

---

Razor components are files written in [Razor](https://learn.microsoft.com/en-us/aspnet/core/mvc/views/razor?view=aspnetcore-8.0) syntax, a templating language that combines HTML and C# code to generate HTML. It serves the same purpose for .NET as JSX does for React.

The introduction of the [RazorComponentResult](https://learn.microsoft.com/en-us/aspnet/core/blazor/components/?view=aspnetcore-8.0) type in .NET 8 has the potential to take our HTMX development to the next level. In our previous post, [Getting Started with HTMX: A Beginner's Guide](https://blog.raulnq.com/getting-started-with-htmx-a-beginners-guide#heading-hx-triggerhttpshtmxorgattributeshx-trigger), we used the `HtmlContentBuilder` class to generate HTML. Now, with the `RazorComponentResult` class, we can harness the power of Razor components to achieve the same goal.

Let's begin migrating our project [here](https://github.com/raulnq/htmx-and-net) to use Razor components. Create the `TodoItemComponent.razor` file to render a single `Todo` item using the following content:

```xml
<div>
    <p>@Model.Name</p>
    @if (Model.IsCompleted)
    {
        <input type="checkbox" checked hx-post="/todos/@Model.Id/toggle" hx-target="closest div" hx-swap="outerHTML" />
    }
    else
    {
        <input type="checkbox" hx-post="/todos/@Model.Id/toggle" hx-target="closest div" hx-swap="outerHTML" />
    }
    <button hx-delete="/todos/@Model.Id" hx-target="closest div" hx-swap="outerHTML">X</button>
</div>
@code {
    [Parameter]
    public TodoItem Model { get; set; } = new();
}
```

The `TodoFormComponent.razor` file will contain the `Form` for adding a new `Todo` item:

```xml
<form hx-post="/todos" hx-swap="beforebegin" hx-ext="json-enc">
    <input type="text" name="name" />
    <button type="submit">Add</button>
</form>
@code {

}
```

The `TodoItemListComponent.razor` file will use the previous components:

```xml
<div>
    @foreach (var item in Model)
    {
        <TodoItemComponent Model="@item"></TodoItemComponent>
    }
    <TodoFormComponent></TodoFormComponent>
</div>
@code {
    [Parameter]
    public List<TodoItem> Model { get; set; } = new();
}
```

And finally, the root component for our application is the `DocumentComponent.razor` file:

```xml
<html>
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width =device-width, initial-scale=1.0">
    <script src="https://unpkg.com/htmx.org@1.9.6" integrity="sha384-FhXw7b6AlE/jyjlZH5iHa/tTe9EpJ1Y55RjcgPbjeWMskSxZt1v9qkxLJWNJaGni" crossorigin="anonymous"></script>
    <script src="https://unpkg.com/htmx.org/dist/ext/json-enc.js"></script>
</head>
<body hx-get="/todos" hx-trigger="load" hx-swap="innerHTML">
</body>
</html>
@code {

}
```

The final `Program.cs` file will be as follows:

```csharp
using Microsoft.AspNetCore.Http.HttpResults;
using TodoApi;
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddRazorComponents();
var app = builder.Build();
var db = new List<TodoItem>()
{
    new TodoItem() { Id = Guid.NewGuid(), Name = "abc", IsCompleted = true },
    new TodoItem() { Id = Guid.NewGuid(), Name = "123", IsCompleted = false },
};
app.MapGet("/", () =>
{
    return new RazorComponentResult<DocumentComponent>();
});
app.MapGet("/todos", () => new RazorComponentResult<TodoItemListComponent>(new { Model = db }));
app.MapPost("/todos/{id}/toggle", (string id) =>
{
    var todo = db.First(t => t.Id.ToString() == id);
    todo.IsCompleted = !todo.IsCompleted;
    return new RazorComponentResult<TodoItemComponent>(new { Model = todo });
});
app.MapDelete("/todos/{id}", (string id) =>
{
    var todo = db.First(t => t.Id.ToString() == id);
    db.Remove(todo);
});
app.MapPost("/todos", (AddTodoItem command) =>
{
    var todo = new TodoItem { Id = Guid.NewGuid(), Name = command.Name, IsCompleted = false };
    db.Add(todo);
    return new RazorComponentResult<TodoItemComponent>(new { Model = todo });
});
app.Run();
```

Razor components allow us to embed C# code in HTML in a clean, readable way, making it easier to design, reuse, and maintain HTML. As a result, implementing our HTMX endpoint becomes much simpler, giving the impression that .NET 8 was designed with HTMX in mind (In fact, it was the support for server-side rendering that made this possible). You can find the final code [here](https://github.com/raulnq/htmx-and-net/tree/razor). Thank you, and happy coding.