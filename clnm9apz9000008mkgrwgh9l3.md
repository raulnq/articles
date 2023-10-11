---
title: "Getting Started with HTMX: A Beginner's Guide"
datePublished: Wed Oct 11 2023 21:21:52 GMT+0000 (Coordinated Universal Time)
cuid: clnm9apz9000008mkgrwgh9l3
slug: getting-started-with-htmx-a-beginners-guide
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1696987355105/62b8a6fc-0415-40e8-82f9-c7871445a695.png
tags: net, htmx

---

HTMX is a JavaScript library that allows you to add AJAX capabilities directly to HTML using HTML attributes. The library is based on two fundamental ideas:

* Allow any HTML element to make an HTTP request.
    
* Use the response HTML to update the element.
    

A basic example might be:

```xml
<button hx-get="/data" hx-trigger="click" hx-swap="outerHTML">
    Get
</button>
```

This button will:

* Make an HTTP GET request to `/data` when clicked.
    
* Replace itself with the response HTML.
    

But why should one consider HTMX when several front-end frameworks are available today? Here are some key points to take into consideration:

* **Simplicity**: HTMX allows developers to build interactive web apps using simple HTML attributes instead of JavaScript code.
    
* **Performance**: Since HTMX relies mainly on HTML (and minimal JavaScript), pages often load faster and use less memory.
    
* **Learning Curve**: Since HTMX works directly with HTML, it has a lower learning curve for developers who are more comfortable with HTML/CSS.
    
* **Server-Side Rendering**: HTMX works with any server-side technology. It allows developers to work in their respective backend languages.
    

## Hello World

Every new technology always starts with the classic "hello world" example. Run the following commands to set up the solution:

```powershell
dotnet new web -o TodoApi
dotnet new sln -n Htmx
dotnet sln add --in-root TodoApi
```

In the `TodoApi` project, and add the `HtmlResult.cs` file with the following content:

```csharp
using System.Net.Mime;
using System.Text;

namespace TodoApi;

public class HtmlResult : IResult
{
    private readonly string _html;

    public HtmlResult(string html)
    {
        _html = html;
    }

    public Task ExecuteAsync(HttpContext httpContext)
    {
        httpContext.Response.ContentType = MediaTypeNames.Text.Html;
        httpContext.Response.ContentLength = Encoding.UTF8.GetByteCount(_html);
        return httpContext.Response.WriteAsync(_html);
    }
}
```

This class will be used to return HTML from our Minimal API endpoints. Go to the `Program.cs` file and update the content as follows:

```csharp
using Microsoft.AspNetCore.Html;
using TodoApi;

var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapGet("/hello-world", () => new HtmlResult(@"<!doctype html>
<html>
    <head>
        <meta charset=""UTF-8"">
        <meta name=""viewport"" content=""width=device-width, initial-scale=1.0"">
        <script src=""https://unpkg.com/htmx.org@1.9.6"" integrity=""sha384-FhXw7b6AlE/jyjlZH5iHa/tTe9EpJ1Y55RjcgPbjeWMskSxZt1v9qkxLJWNJaGni"" crossorigin=""anonymous""></script>
    </head>
    <body>
        <button hx-post=""/hello-world"" hx-swap=""outerHTML"">Get time</button>
    </body>
</html>
"));
app.MapPost("/hello-world", () => new HtmlResult($@"<div>Hello World {DateTimeOffset.UtcNow}</div>"));
app.Run();
```

There are two endpoints in our API. The first one returns the initial HTML document, which includes a reference to the HTMX library via CDN. Within the body, there is a button with the HTMX attributes (to be explained later) that instructs the browser to make an HTTP POST request to `/hello-world` when the button is clicked and then replaces the button with the response. The second endpoint returns only a div tag containing the message. To see this in action, run the application and navigate to [http://localhost:5179/hello-world](http://localhost:5179/hello-world).

## Attributes

The primary attributes used in HTMX are designed for making AJAX requests:

* [hx-get](https://htmx.org/attributes/hx-get/): Issues a GET request to the given URL.
    
* [hx-post](https://htmx.org/attributes/hx-post/): Issues a POST request to the given URL.
    
* [hx-put](https://htmx.org/attributes/hx-put/): Issues a PUT request to the given URL.
    
* [hx-patch](https://htmx.org/attributes/hx-patch/): Issues a PATCH request to the given URL.
    
* [hx-delete](https://htmx.org/attributes/hx-delete/): Issues a DELETE request to the given URL.
    

The following attributes are most commonly used. For more information, please visit the official website [here](https://htmx.org/).

### [hx-trigger](https://htmx.org/attributes/hx-trigger/)

The hx-trigger attribute enables us to define what initiates an AJAX request. By default, there is no need to specify the attribute when the following triggers are used:

* `input`, `textarea` & `select` are triggered on the `change` event.
    
* `form` is triggered on the `submit` event.
    
* everything else is triggered by the `click` event.
    

A trigger value can be one of the following:

* An event name.
    
* A polling definition.
    
* A comma-separated list of such events
    

### [hx-target](https://htmx.org/attributes/hx-target/)

The `hx-target` attribute enables us to target a different element for swapping, rather than the one initiating the AJAX request. The value of this attribute can be:

* A CSS query selector of the element to target.
    
* `this` indicates that the element with the `hx-target` attribute is the target itself.
    
* `closest <CSS selector>` finds the closest ancestor element or itself, that matches the given CSS selector.
    
* `find <CSS selector>` finds the first child descendant element that matches the given CSS selector.
    
* `next <CSS selector>` scans the DOM forward for the first element that matches the given CSS selector.
    
* `previous <CSS selector>` scans the DOM backward for the first element that matches the given CSS selector.
    

### [hx-swap](https://htmx.org/attributes/hx-swap/)

The `hx-swap` attribute allows you to specify how the response will be swapped relative to the target of an AJAX request. The possible values of this attribute are:

* `innerHTML`: Puts the content inside the target element. The default option.
    
* `outerHTML`: Replace the entire target element with the response.
    
* `beforebegin`: Insert the content before the target in the parent element.
    
* `afterbegin`: Insert the response before the first child of the target element.
    
* `beforeend`: Insert the response after the last child of the target element.
    
* `afterend`: Insert the response after the target in the parent element.
    
* `delete`: Deletes the target element regardless of the response.
    
* `none`: Does not append content from the response.
    

## Todo App

Let's build an application to see all these concepts in action. To build HTML in .NET we can use the `HtmlContentBuilder` class rather than using a simple string. This has several benefits:

* **Parameterizable**: Since `HtmlContentBuilder` uses method calls, you can pass parameters to those methods to dynamically build up HTML.
    
* **Supports Encoding**: `HtmlContentBuilder` will automatically HTML encode any unencoded strings you append.
    
* **Composable**: We can compose multiple `HtmlContentBuilder` instances to build up more complex HTML.
    
* **Easier to Maintain**: Building HTML as a single string can become difficult to maintain as the HTML grows more complex. `HtmlContentBuilder` allows you to build up the HTML incrementally using method calls.
    

Let's create a new `IResult` implementation based on the `IHtmlContent` interface:

```csharp
using Microsoft.AspNetCore.Html;
using System.Net.Mime;
using System.Text.Encodings.Web;
using System.Text;
namespace TodoApi;
public class HtmlContentResult : IResult
{
    private readonly IHtmlContent _htmlContent;

    public HtmlContentResult(IHtmlContent htmlContent)
    {
        _htmlContent = htmlContent;
    }

    public Task ExecuteAsync(HttpContext httpContext)
    {
        httpContext.Response.ContentType = MediaTypeNames.Text.Html;
        using (var writer = new StringWriter())
        {
            _htmlContent.WriteTo(writer, HtmlEncoder.Default);
            var html = writer.ToString();
            httpContext.Response.ContentLength = Encoding.UTF8.GetByteCount(html);
            return httpContext.Response.WriteAsync(html);
        }
    }
}
```

Create a `TodoItem.cs` file to store our model:

```csharp
namespace TodoApi;
public class TodoItem
{
    public Guid Id { get; set; }
    public bool IsCompleted { get; set; }
    public string? Name { get; set; }
};
```

Create a `Components.cs` class to contain all of our HTML builders:

```csharp
using Microsoft.AspNetCore.Html;
namespace TodoApi;
public static class Components
{
    public static IHtmlContent Document(IHtmlContentContainer children)
    {
        var builder = new HtmlContentBuilder();
        builder.AppendHtml("<!doctype html>");
        builder.AppendHtml("<html>");
        builder.AppendHtml("<head>");
        builder.AppendHtml("<meta charset=\"UTF-8\">");
        builder.AppendHtml("<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">");
        builder.AppendHtml("<script src=\"https://unpkg.com/htmx.org@1.9.6\" integrity=\"sha384-FhXw7b6AlE/jyjlZH5iHa/tTe9EpJ1Y55RjcgPbjeWMskSxZt1v9qkxLJWNJaGni\" crossorigin=\"anonymous\"></script>");
        builder.AppendHtml("</head>");
        builder.AppendHtml(children);
        builder.AppendHtml("</html>");
        return builder;
    }

    public static IHtmlContent TodoList(IEnumerable<TodoItem> todoItems)
    {
        var builder = new HtmlContentBuilder();
        builder.AppendHtml("<div>");
        foreach (var todoItem in todoItems)
        {
            builder.AppendHtml(TodoItem(todoItem));
        }
        builder.AppendHtml("</div>");
        return builder;
    }

    public static IHtmlContent TodoItem(TodoItem todoItem)
    {
        var builder = new HtmlContentBuilder();
        builder.AppendHtml("<div>");
        builder.AppendHtml("<p>");
        builder.AppendFormat("{0}", todoItem.Name);
        builder.AppendHtml("</p>");
        builder.AppendHtml("</div>");
        return builder;
    }
}
```

Navigate to the `Program.cs` file and add the following code:

```csharp
var db = new List<TodoItem>()
{
    new TodoItem() { Id = Guid.NewGuid(), Name = "abc", IsCompleted = true },
    new TodoItem() { Id = Guid.NewGuid(), Name = "123", IsCompleted = false },
};

app.MapGet("/", () =>
{
    var builder = new HtmlContentBuilder();
    builder.AppendHtml("<body hx-get=\"/todos\" hx-trigger=\"load\" hx-swap=\"innerHTML\">");
    builder.AppendHtml("</body>");
    return new HtmlContentResult(Components.Document(builder));
});

app.MapGet("/todos", () => new HtmlContentResult(Components.TodoList(db)));
```

The first endpoint renders the initial HTML document, while the second returns a list of items. During the body tag's load event (`hx-trigger="load"`), HTMX makes a request (`hx-get="/todos"`) and inserts the response inside the element (`hx-swap="innerHTML"`). Let's add the features to toggle and delete items. Update the `TodoItem` component as follows:

```csharp
    public static IHtmlContent TodoItem(TodoItem todoItem)
    {
        var isCompleted = string.Empty;
        if (todoItem.IsCompleted)
        {
            isCompleted = "checked";
        }
        var builder = new HtmlContentBuilder();
        builder.AppendHtml("<div>");
        builder.AppendHtml("<p>");
        builder.AppendFormat("{0}", todoItem.Name);
        builder.AppendHtml("</p>");
        builder.AppendHtml($"<input type=\"checkbox\" {isCompleted} hx-post=\"/todos/{todoItem.Id}/toggle\" hx-target=\"closest div\" hx-swap=\"outerHTML\"/>");
        builder.AppendHtml($"<button hx-delete=\"/todos/{todoItem.Id}\" hx-target=\"closest div\" hx-swap=\"outerHTML\">X</button>");
        builder.AppendHtml("</div>");
        return builder;
    }
```

During the change event of the checkbox input, HTMX initiates a request (`hx-post="/todos/{todoItem.Id}/toggle"`) and replaces the element (`hx-target="closest div"`) with the response (`hx-swap="outerHTML"`), updating the list. In the same line, the click event performs a similar action; however, since the response is empty, the item is removed from the list. In the `Program.cs` file, add these two endpoints:

```csharp
app.MapPost("/todos/{id}/toggle", (string id) =>
{
    var todo = db.First(t => t.Id.ToString() == id);
    todo.IsCompleted = !todo.IsCompleted;
    return new HtmlContentResult(Components.TodoItem(todo));
});

app.MapDelete("/todos/{id}", (string id) =>
{
    var todo = db.First(t => t.Id.ToString() == id);
    db.Remove(todo);
});
```

The final feature will enable the addition of new items to the list. Navigate to the `TodoItem.cs` file and append the following code:

```csharp
record AddTodoItem(string? Name);
```

Next, in the `Program.cs` file, add the following endpoint:

```csharp
app.MapPost("/todos", (AddTodoItem command) =>
{
    var todo = new TodoItem {Id = Guid.NewGuid(),Name = command.Name, IsCompleted = false };
    db.Add(todo);
    return new HtmlContentResult(Components.TodoItem(todo));
});
```

Navigate to the `Components.cs` file and modify the `Document` component to install the `json-enc` [extension](https://htmx.org/docs/#extensions) to encode parameters in JSON format instead of URL format:

```csharp
    public static IHtmlContent Document(IHtmlContentContainer children)
    {
        var builder = new HtmlContentBuilder();
        builder.AppendHtml("<!doctype html>");
        builder.AppendHtml("<html>");
        builder.AppendHtml("<head>");
        builder.AppendHtml("<meta charset=\"UTF-8\">");
        builder.AppendHtml("<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">");
        builder.AppendHtml("<script src=\"https://unpkg.com/htmx.org@1.9.6\" integrity=\"sha384-FhXw7b6AlE/jyjlZH5iHa/tTe9EpJ1Y55RjcgPbjeWMskSxZt1v9qkxLJWNJaGni\" crossorigin=\"anonymous\"></script>");
        builder.AppendHtml("<script src=\"https://unpkg.com/htmx.org/dist/ext/json-enc.js\"></script>");
        builder.AppendHtml("</head>");
        builder.AppendHtml(children);
        builder.AppendHtml("</html>");
        return builder;
    }
```

In the same file add a `TodoForm` component:

```csharp
    public static IHtmlContent TodoForm()
    {
        var builder = new HtmlContentBuilder();
        builder.AppendHtml("<form hx-post=\"/todos\" hx-swap=\"beforebegin\" hx-ext=\"json-enc\">");
        builder.AppendHtml("<input type=\"text\" name=\"name\">");
        builder.AppendHtml("<button type=\"submit\">Add</button>");
        builder.AppendHtml("</form>");
        return builder;
    }
```

During the submit event, HTMX initiates a request (`hx-post="/todos"`) in JSON format (`hx-ext="json-enc"`) and appends the response in the parent element before the form (`hx-swap="beforebegin"`). Finally, modify the `TodoList` component to include the `TodoForm` component:

```csharp
    public static IHtmlContent TodoList(IEnumerable<TodoItem> todoItems)
    {
        var builder = new HtmlContentBuilder();
        builder.AppendHtml("<div>");
        foreach (var todoItem in todoItems)
        {
            builder.AppendHtml(TodoItem(todoItem));
        }
        builder.AppendHtml(TodoForm());
        builder.AppendHtml("</div>");
        return builder;
    }
```

In conclusion, HTMX is a powerful and lightweight JavaScript library that simplifies the process of building interactive web applications. By utilizing HTML attributes to make AJAX requests, HTMX offers an easy learning curve, improved performance, and seamless integration with server-side technologies. This beginner's guide provides a solid foundation for getting started with HTMX, showcasing its capabilities through a simple Todo App example. All the code is available [here](https://github.com/raulnq/htmx-and-net). Thanks, and happy coding.