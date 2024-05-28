---
title: "Developing Your First App with HTMX and .NET: Part I"
datePublished: Tue May 28 2024 00:08:28 GMT+0000 (Coordinated Universal Time)
cuid: clwpn31yb000008jn77yab6t8
slug: developing-your-first-app-with-htmx-and-net-part-i
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1716820224082/1d4a5e3b-2d44-49eb-a1a2-94f6f5b2e748.png
tags: net, htmx

---

> htmx gives you access to AJAX, CSS Transitions, WebSockets and Server Sent Events directly in HTML, using attributes, so you can build modern user interfaces with the simplicity and power of hypertext

[HTMX](https://htmx.org/docs/) is gaining attention for its simplicity and lightweight nature, making it attractive to developers who want to enhance their web applications without the complexity of frameworks like React or Angular. Additionally, integrating it with ASP.NET is easier than ever, as we saw in the article [.NET 8: Working with HTMX and Razor Components](https://blog.raulnq.com/net-8-working-with-htmx-and-razor-components), making it a viable alternative for our developments. Due to this, we decided to write a series of articles to explore different aspects of the library while building a simple web application to handle products, so let's get started.

## Pre-requisites

* Install [Docker Desktop](https://docs.docker.com/desktop/install/windows-install/)
    

## Database Setup

Run the following command to create a new container with our database:

```powershell
docker run --detach --env ACCEPT_EULA=Y --env MSSQL_SA_PASSWORD=Sqlserver123$ --name htmx-sqlserver --publish 1433:1433 mcr.microsoft.com/mssql/server:2019-CU14-ubuntu-20.04
```

Run the following script using your favorite SQL Server client:

```sql
USE master;
GO
CREATE DATABASE MyApp
GO
USE MyApp;
GO
CREATE TABLE dbo.[Products] (
    [ProductId] UNIQUEIDENTIFIER NOT NULL,
    [Name] nvarchar(100) NOT NULL,
	[Description] nvarchar(500) NULL,
	[Unit] nvarchar(20) NOT NULL,
    [Price] decimal(19,4) NOT NULL,
    [IsEnabled] bit NOT NULL,
	[DeletedAt] datetimeoffset NULL,
    [DeletionReason] nvarchar(500) NULL,
    CONSTRAINT [PK_Products] PRIMARY KEY ([ProductId])
);
GO
```

## Solution Setup

Run the following commands to set up the solution:

```powershell
dotnet new web -o MyApp
dotnet new sln -n MyApp
dotnet sln add --in-root MyApp
dotnet add MyApp package Microsoft.EntityFrameworkCore.SqlServer
```

Open the solution to add our first [Razor component](https://learn.microsoft.com/en-us/aspnet/core/blazor/components/?view=aspnetcore-8.0). Create the `MainPage.razor` file with the following content:

```xml
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width =device-width, initial-scale=1.0">
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/css/bootstrap.min.css" integrity="sha384-QWTKZyjpPEjISv5WaRU9OFeRpok6YctnYmDr5pNlyT2bRjXh0JMhjY6hW+ALEwIH" crossorigin="anonymous" />
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap-icons@1.11.3/font/bootstrap-icons.min.css" />
</head>
<body class="mb-5">
    <header class="navbar navbar-expand bg-white fixed-top border-bottom px-4 z-2" style="height: 3.875rem;">
        <div class="container-fluid">
            <div class="collapse navbar-collapse">
                <ul class="navbar-nav ms-auto">
                    <li class="nav-item">
                        <span>Welcome!</span>
                    </li>
                </ul>
            </div>
        </div>
    </header>
    <aside class="navbar navbar-expand z-3 d-block position-fixed top-0 start-0 bottom-0 border-end bg-white p-0 ms-0" style="width: 16.25rem;">
        <a class="navbar-brand py-0 px-4 d-flex align-items-center" href="#" aria-label="Front" style="height: 3.875rem;">
            <img class="d-block" src="https://placehold.co/100x50" alt="Logo" style="min-width: 6.5rem; max-width: 6.5rem;">
        </a>
        <div class="overflow-y-auto" style="height: calc(100% - 3.875rem);">
            <div class="d-flex flex-column px-4">
                <span class="mt-4 px-3 py-2 fw-bold fs-6">MENU</span>
                <ul class="nav nav-pills">
                    <li class="nav-item">
                        <a class="nav-link link-dark" href="#">
                            <span>Option</span>
                        </a>
                    </li>
                </ul>
            </div>
        </div>
    </aside>
    <main style="padding-left: 16.25rem; padding-top:3.875rem">
        <div class="container-fluid p-4">
        </div>
    </main>
    <script src="https://unpkg.com/htmx.org@1.9.6" integrity="sha384-FhXw7b6AlE/jyjlZH5iHa/tTe9EpJ1Y55RjcgPbjeWMskSxZt1v9qkxLJWNJaGni" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/bootstrap@5.3.3/dist/js/bootstrap.bundle.min.js" integrity="sha384-YvpcrYf0tY3lHB60NNkmXc5s9fDVZLESaAA55NDzOxhy9GkcIdslK1eN7N6jIeHz" crossorigin="anonymous"></script>
</body>
</html>
@code {
}
```

This file will serve as the initial layout for our application. It consists of three sections: a navigation bar at the top, a sidebar on the left, and a main area on the right. We are also adding references to [Bootstrap](https://getbootstrap.com/docs/5.3/getting-started/introduction/) and HTMX via CDN. Open the `Program.cs` file and update the content as follows:

```csharp
using Microsoft.AspNetCore.Http.HttpResults;
using MyApp;
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddRazorComponents();
builder.Services.ConfigureHttpJsonOptions(options =>
{
    options.SerializerOptions.Converters.Add(new JsonStringEnumConverter());
});
var app = builder.Build();
app.MapGet("/", () =>
{
    return new RazorComponentResult<MainPage>();
});
app.Run();
```

The `AddRazorComponents` method configures the services needed to render Razor components. Additionally, we define an endpoint to return the previous component.

## Entity Framework Setup

Create the `MyAppDbContext.cs` file with the following content:

```csharp
using Microsoft.EntityFrameworkCore;
using System.Reflection;

namespace MyApp;

public class MyAppDbContext : DbContext
{
    public MyAppDbContext(DbContextOptions options) : base(options)
    {
    }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfigurationsFromAssembly(Assembly.GetExecutingAssembly());
        base.OnModelCreating(modelBuilder);
    }
}
```

The `ApplyConfigurationsFromAssembly` method will apply all the implementations of the `IEntityTypeConfiguration<TEntity>` interface in our `DbContext`. Update the `appsettings.json` file as follows:

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*",
  "ConnectionString": "Server=localhost,1433;Database=MyApp;User ID=sa;Password=Sqlserver123$;MultipleActiveResultSets=true;TrustServerCertificate=True;"
}
```

We are adding the `ConnectionString` pointing to our database. Add the following line to the `Program.cs` file:

```csharp
builder.Services.AddDbContext<MyAppDbContext>(options => options.UseSqlServer(builder.Configuration["ConnectionString"]));
```

## The Model

Create a `Products/Product.cs` file as follows:

```csharp
namespace MyApp.Products;
public class Product
{
    public Guid ProductId { get; set; }
    public string Name { get; set; } = default!;
    public string? Description { get; set; }
    public decimal Price { get; set; }
    public bool IsEnabled { get; set; }
    public Unit Unit { get; set; }
    public DateTimeOffset? DeletedAt { get; set; }
    public string? DeletionReason { get; set; }
}
public enum Unit
{
    UN,
    KG
}
```

Create a `Products/EntityTypeConfiguration.cs` file as follows:

```csharp
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using Microsoft.EntityFrameworkCore;
namespace MyApp.Products;
public class EntityTypeConfiguration : IEntityTypeConfiguration<Product>
{
    public void Configure(EntityTypeBuilder<Product> builder)
    {
        builder
            .ToTable("Products");
        builder
            .HasKey(cr => cr.ProductId);
        builder
            .Property(c => c.Price)
            .HasColumnType("decimal(19, 4)");
        builder
            .Property(c => c.Unit)
            .HasConversion(s => s.ToString(), value => (Unit)Enum.Parse(typeof(Unit), value, true));
    }
}
```

## List Products

Create a `Products/ListProducts.cs` file with:

```csharp
using Microsoft.AspNetCore.Http.HttpResults;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
namespace MyApp.Products;
public static class ListProducts
{
    public class Request
    {
        public string? Name { get; set; }
    }

    public static async Task<RazorComponentResult> HandlePage([FromServices] MyAppDbContext appDbContext, [AsParameters] Request request)
    {
        var name = request.Name ?? string.Empty;
        var results = await appDbContext.Set<Product>().AsNoTracking().Where(p => p.Name.Contains(name)).ToListAsync();
        return new RazorComponentResult<ListProductsPage>(new { Results = results });
    }
}
```

The `HandlePage` method processes the request and returns the corresponding Razor component. Create a `Products/Endpoints.cs` file with the following content:

```csharp
namespace MyApp.Products;
public static class Endpoints
{
    public static void RegisterProductEndpoints(this WebApplication app)
    {
        var group = app.MapGroup("/products");
        group.MapGet("/list", ListProducts.HandlePage);
    }
}
```

Add the following line to the `Program.cs` file to register our new endpoint:

```csharp
app.RegisterProductEndpoints();
```

Create a `Products/ListProductsPage.razor` file with the following content:

```xml
<h4>List Products</h4>
<nav class="navbar hstack gap-3 justify-content-end">
</nav>
<div class="card mb-4">
    <div class="card-body">
        <div class="row mb-4">
            <div class="col">
                <input type="search"
                       class="form-control"
                       name="Name"
                       placeholder="Enter name"
                       hx-trigger="input changed delay:500ms, search"
                       hx-get="/products/list"
                       hx-swap="OuterHTML"
                       hx-target="#results"
                       hx-select="#results" />
            </div>
        </div>
        <div id="results">
            <div class="table-responsive">
                <table class="table">
                    <thead class="table-light">
                        <tr>
                            <th>#</th>
                            <th>Name</th>
                            <th>Unit</th>
                            <th>Price</th>
                            <th>Is Enabled</th>
                        </tr>
                    </thead>
                    <tbody>
                        @foreach (var item in Results)
                        {
                            <tr>
                                <td>@item.ProductId</td>
                                <td>@item.Name</td>
                                <td>@item.Unit</td>
                                <td>@item.Price</td>
                                <td>@item.IsEnabled</td>
                            </tr>
                        }
                    </tbody>
                </table>
            </div>
        </div>
    </div>
</div>
@code {
    [Parameter]
    public List<Product> Results { get; set; } = default!;
}
```

In this Razor component, we can identify three main sections:

* A navigation bar in the top section, which will be used later.
    
* A section containing the filters for our search. The HTMX attributes used here are:
    
    * [`hx-get`](https://htmx.org/attributes/hx-get/): Specifies where the element will send the AJAX request. In this case, we are sending the request to `/products/list`.
        
    * [`hx-trigger`](https://htmx.org/attributes/hx-trigger/): Specifies what triggers the AJAX request in the element. The request will be triggered by an input change with a 500-millisecond debounce or by the search event.
        
    * [`hx-target`](https://htmx.org/attributes/hx-target/): Specifies where the response will be swapped in. We are targeting the element with the ID `results`, which is the container of the table.
        
    * [`hx-swap`](https://htmx.org/attributes/hx-swap/): Specifies how to swap the response in the target. The `OuterHTML` replaces the entire target element with the response.
        
    * [`hx-select`](https://htmx.org/attributes/hx-select/): Specifies the content we want to be swapped from the response. We will only select the element's content with the ID `results` from the response.
        
* A table to display the results of our search.
    

Go to the `MainPage.razor` file and replace the `<li>` element with:

```xml
<li class="nav-item">
	<a class="nav-link link-dark"
	   href="#"
	   hx-get="/products/list"
	   hx-target="#main"
	   hx-swap="innerHTML">
		<span>Products</span>
	</a>
</li>
```

With this setup, the link will render the `/products/list` response inside the element with the ID `main`.

## Register Product

Create a `Products/RegisterProduct.cs` file with:

```csharp
using Microsoft.AspNetCore.Http.HttpResults;
using Microsoft.AspNetCore.Mvc;
namespace MyApp.Products;
public static class RegisterProduct
{
    public class Request
    {
        public string Name { get; set; } = default!;
        public string? Description { get; set; }
        public decimal Price { get; set; }
        public Unit Unit { get; set; }
    }

    public static Task<RazorComponentResult> HandlePage()
    {
        return Task.FromResult<RazorComponentResult>(new RazorComponentResult<RegisterProductPage>());
    }

    public static async Task<RazorComponentResult> HandleAction([FromServices] MyAppDbContext appDbContext, [FromBody] Request request)
    {
        var product = new Product()
        {
            ProductId = Guid.NewGuid(),
            Name = request.Name,
            Description = request.Description,
            Price = request.Price,
            Unit = request.Unit,
            IsEnabled = false
        };
        appDbContext.Set<Product>().Add(product);
        await appDbContext.SaveChangesAsync();
        return await ListProducts.HandlePage(appDbContext, new ListProducts.Request());
    }
}
```

The `HandlePage` method will receive the request to render the registration form, and the `HandleAction` method will process the resulting request from the form. Update the `Endpoints.cs` file as follows:

```csharp
namespace MyApp.Products;
public static class Endpoints
{
    public static void RegisterProductEndpoints(this WebApplication app)
    {
        var group = app.MapGroup("/products");
        group.MapGet("/list", ListProducts.HandlePage);
        group.MapGet("/register", RegisterProduct.HandlePage);
        group.MapPost("/register", RegisterProduct.HandleAction);
    }
}
```

Modify the `Products/ListProductsPage.razor` file by replacing the `<nav>` element with:

```xml
<nav class="navbar hstack gap-3 justify-content-end">
    <button type="button"
            hx-get="/products/register"
            hx-target="#main"
            hx-swap="innerHTML"
            class="btn btn-primary">
        New
    </button>
</nav>
```

We are adding the button to render the `/products/register` response inside the element with the ID `main`. Create a `Products/RegisterProductPage.razor` file with the following content:

```xml
<h4>Register Products</h4>
<div class="card mb-4">
    <div class="card-body">
        <form hx-post="/products/register"
              hx-target="#main"
              hx-swap="innerHTML"
              hx-ext="json-enc">
            <div class="row mb-4">
                <div class="col-4">
                    <div class="form-group">
                        <label for="Name" class="form-label">Name</label>
                        <input type="text"
                               class="form-control"
                               id="Name"
                               name="Name"
                               placeholder="Enter name"
                               maxlength=100
                               required />
                    </div>
                </div>
                <div class="col-4">
                    <div class="form-group">
                        <label for="Price" class="form-label">Price</label>
                        <div class="input-group">
                            <span class="input-group-text">$</span>
                            <input type="number"
                                   class="form-control text-end"
                                   id="Price"
                                   name="Price"
                                   step="0.01"
                                   min="0.01"
                                   value="0"
                                   required />
                        </div>
                    </div>
                </div>
                <div class="col-4">
                    <div class="form-group">
                        <label for="Unit" class="form-label">Unit</label>
                        <select class="form-select"
                                name="Unit"
                                id="Unit"
                                required>
                            <option value="UN" selected>Unit</option>
                            <option value="KG">Kilogram</option>
                        </select>
                    </div>
                </div>
            </div>
            <div class="row mb-4">
                <div class="col">
                    <div class="form-group">
                        <label for="Description" class="form-label">Description</label>
                        <textarea class="form-control"
                                  id="Description"
                                  name="Description"
                                  rows="5"
                                  placeholder="Enter description"
                                  maxlength=500 />
                    </div>
                </div>
            </div>
            <button type="submit" class="btn btn-primary">
                <span>Save Changes</span>
            </button>
        </form>
    </div>
</div>
@code {
}
```

A new HTMX attribute is used here, [`hx-ext`](https://htmx.org/attributes/hx-ext/), which enables an [extension](https://htmx.org/extensions/) for an element. In this case, the [extension](https://htmx.org/extensions/json-enc/) encodes the payload in `JSON` format. To complete the setup, add the following script in the `MainPage.razor` file:

```xml
<script src="https://unpkg.com/htmx.org/dist/ext/json-enc.js"></script>
```

Run the application, start to register products, and filter the results.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1716854723796/017e4ef8-d9ee-49dc-b26b-84204befb1d0.png align="center")

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1716854748007/254908d7-1043-4806-bbe9-a7c8cb8748d1.png align="center")

You can find the final code [here](https://github.com/raulnq/htmx-dotnet-app/tree/part-i). Thank you, and happy coding.