---
title: "Developing Your First App with HTMX and .NET: Part II"
datePublished: Sat Jun 01 2024 14:15:35 GMT+0000 (Coordinated Universal Time)
cuid: clww73unv000009mea8z0bb5u
slug: developing-your-first-app-with-htmx-and-net-part-ii
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1717215955789/b3f87470-aeaf-4ff1-9246-09b7cb5ac211.png
tags: net, dotnet, htmx

---

A new episode in our [HTMX and .NET series](https://blog.raulnq.com/series/htmx) has arrived. This time, we will be adding an edit feature to our app. Before diving into that, we'll refactor our code by creating several Razor components to enhance code reusability. You can download the starting code from the previous [article](https://blog.raulnq.com/developing-your-first-app-with-htmx-and-net-part-i).

## ListProductsPage.razor

So, let's start by refactoring the `ListProductsPage.razor` file. Create the `RazorComponents/DataTable.razor` file with the following content:

```xml
@typeparam TItem
<div class="table-responsive">
    <table class="table">
        <thead class="table-light">
            <tr>@TableHeader</tr>
        </thead>
        <tbody>
            @if (Items is not null && Items.Any())
            {
                @foreach (var item in Items)
                {
                    <tr>
                        @RowTemplate(item)
                    </tr>
                }
            }
        </tbody>
    </table>
</div>
@code {
    [Parameter, EditorRequired]
    public IEnumerable<TItem> Items { get; set; } = default!;
    [Parameter, EditorRequired]
    public RenderFragment<TItem> RowTemplate { get; set; } = default!;
    [Parameter, EditorRequired]
    public RenderFragment? TableHeader { get; set; } = default!;
}
```

This [templated component](https://learn.microsoft.com/en-us/aspnet/core/blazor/components/templated-components?view=aspnetcore-8.0) will display a list of items. Create the `RazorComponents/HtmxAttributes.cs` file with the following content:

```csharp
namespace MyApp.RazorComponents;

public class HtmxAttributes
{
    public string Target { get; set; } = default!;
    public string Swap { get; set; } = "innerHTML";
    public string Endpoint { get; set; } = default!;
    public string Select { get; set; } = default!;
    public HtmxAttributes()
    {
    }

    public HtmxAttributes(string endpoint, string target)
    {
        Endpoint = endpoint;
        Target = target;
    }
    public HtmxAttributes(string endpoint, string target, string swap) : this(endpoint, target)
    {
        Swap = swap;
    }

    public HtmxAttributes(string endpoint, string target, string swap, string select) : this(endpoint, target, swap)
    {
        Select = select;
    }
}
```

The class `HtmxAttributes` will serve as a parameter for components with HTMX tags. Create the `RazorComponents/SearchFilter.razor` as follows:

```xml
<input type="search"
       class="form-control"
       name=@Property
       hx-trigger="input changed delay:500ms, search"
       hx-get=@HtmxAttributes.Endpoint
       hx-swap=@HtmxAttributes.Swap
       hx-target=@HtmxAttributes.Target
       hx-select=@HtmxAttributes.Select
       @attributes="Attributes" />
@code {
    [Parameter, EditorRequired]
    public string Property { get; set; } = default!;
    [Parameter(CaptureUnmatchedValues = true)]
    public Dictionary<string, object>? Attributes { get; set; }
    [Parameter, EditorRequired]
    public HtmxAttributes HtmxAttributes { get; set; } = default!;
}
```

The `Attributes` parameter is used to [capture unexpected parameters](https://learn.microsoft.com/en-us/aspnet/core/blazor/components/splat-attributes-and-arbitrary-parameters?view=aspnetcore-8.0) in our component. Create the `RazorComponents/ActionButton.razor` file as follows:

```xml
<button type="button"
        hx-get=@HtmxAttributes.Endpoint
        hx-target=@HtmxAttributes.Target
        hx-swap=@HtmxAttributes.Swap
        class="@Icon btn btn-primary">
    New
</button>
@code {
    [Parameter, EditorRequired]
    public string Label { get; set; } = default!;
    [Parameter, EditorRequired]
    public HtmxAttributes HtmxAttributes { get; set; } = default!;
    [Parameter]
    public string Icon { get; set; } = string.Empty;
}
```

Create the `RazorComponents/Section.razor` file with the following content:

```xml
<div class="card mb-4">
    <div class="card-body">
        @Content
    </div>
</div>
@code {
    [Parameter, EditorRequired]
    public RenderFragment? Content { get; set; } = default!;
}
```

The `ListProductsPage.razor` file using the new components will look like this:

```xml
@using MyApp.RazorComponents;
<h4>List Products</h4>
<nav class="navbar hstack gap-3 justify-content-end">
    <ActionButton Label="New"
                  Icon="bi bi-plus-lg"
                  HtmxAttributes=@(new HtmxAttributes("/products/register", "#main", "innerHTML")) />
</nav>
<Section>
    <Content>
        <div class="row mb-4">
            <div class="col">
                <SearchFilter Property="Name"
                              placeholder="Enter name"
                              HtmxAttributes=@(new HtmxAttributes("/products/list", "#results", "OuterHTML", "#results")) />

            </div>
        </div>
        <div id="results">
            <DataTable Items=@Results
                       Context="item">
                <TableHeader>
                    <th>#</th>
                    <th>Name</th>
                    <th>Unit</th>
                    <th>Price</th>
                    <th>Is Enabled</th>
                </TableHeader>
                <RowTemplate>
                    <td>@item.ProductId</td>
                    <td>@item.Name</td>
                    <td>@item.Unit</td>
                    <td>@item.Price</td>
                    <td>@item.IsEnabled</td>
                </RowTemplate>
            </DataTable>
        </div>
    </Content>
</Section>
@code {
    [Parameter]
    public List<Product> Results { get; set; } = default!;
}
```

To use the [icon](https://icons.getbootstrap.com/) `bi bi-plus-lg`, add the following `link` tag in the `MainPage.razor` file:

```xml
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap-icons@1.11.3/font/bootstrap-icons.min.css" />
```

## RegisterProductPage.razor

Let's do the same exercise with the file `RegisterProductPage.razor`. The new components will be:

`RazorComponents/TextInput.razor`

```xml
<div class="form-group">
    <label for=@Property class="form-label">@Label</label>
    <input type="text"
           class="form-control"
           id=@Property
           name=@Property
           @attributes="Attributes" />
</div>
@code {
    [Parameter, EditorRequired]
    public string Property { get; set; } = default!;
    [Parameter, EditorRequired]
    public string Label { get; set; } = default!;
    [Parameter(CaptureUnmatchedValues = true)]
    public Dictionary<string, object>? Attributes { get; set; }
}
```

`RazorComponents/TextAreaInput.razor`

```xml
<div class="form-group">
    <label for=@Property class="form-label">@Label</label>
    <textarea class="form-control"
              id=@Property
              name=@Property
              @attributes="Attributes" />
</div>
@code {
    [Parameter, EditorRequired]
    public string Property { get; set; } = default!;
    [Parameter, EditorRequired]
    public string Label { get; set; } = default!;
    [Parameter(CaptureUnmatchedValues = true)]
    public Dictionary<string, object>? Attributes { get; set; }
}
```

`RazorComponents/SelectInput.razor`

```xml
@typeparam TKey where TKey : notnull
<div class="form-group">
    <label for=@Property class="form-label">@Label</label>
    <select class="form-select"
            name=@Property
            id=@Property
            @attributes="Attributes">

        @foreach (var item in Source)
        {
            @if (Value != null && Value.Equals(item.Key))
            {
                <option value=@item.Key selected>@item.Value</option>
            }
            else
            {
                <option value=@item.Key>@item.Value</option>
            }
        }
    </select>
</div>
@code {
    [Parameter, EditorRequired]
    public string Property { get; set; } = default!;
    [Parameter, EditorRequired]
    public string Label { get; set; } = default!;
    [Parameter]
    public TKey Value { get; set; } = default!;
    [Parameter]
    public Dictionary<TKey, string> Source { get; set; } = new Dictionary<TKey, string>();
    [Parameter(CaptureUnmatchedValues = true)]
    public Dictionary<string, object>? Attributes { get; set; }
}
```

`RazorComponents/NumericInput.razor`

```xml
<div class="form-group">
    <label for=@Property class="form-label">@Label</label>
    @if (string.IsNullOrEmpty(Prefix) && string.IsNullOrEmpty(Sufix))
    {
        <input type="number"
               class="form-control"
               id=@Property
               name=@Property
               @attributes="Attributes" />
    }
    else
    {
        <div class="input-group">
            @if (!string.IsNullOrEmpty(Prefix))
            {
                <span class="input-group-text">@Prefix</span>
            }
            <input type="number"
                   class="form-control"
                   id=@Property
                   name=@Property
                   @attributes="Attributes" />
            @if (!string.IsNullOrEmpty(Sufix))
            {
                <span class="input-group-text">@Sufix</span>
            }
        </div>
    }
</div>
@code {
    [Parameter, EditorRequired]
    public string Property { get; set; } = default!;
    [Parameter, EditorRequired]
    public string Label { get; set; } = default!;
    [Parameter(CaptureUnmatchedValues = true)]
    public Dictionary<string, object>? Attributes { get; set; }
    [Parameter]
    public string Prefix { get; set; } = default!;
    [Parameter]
    public string Sufix { get; set; } = default!;
}
```

`RazorComponents/Form.razor`

```xml
<form hx-post=@HtmxAttributes.Endpoint
      hx-target=@HtmxAttributes.Target
      hx-swap=@HtmxAttributes.Swap
      hx-ext="json-enc">
    @Content
    <button type="submit" class="btn btn-primary">
        <span>Save Changes</span>
    </button>
</form>
@code {
    [Parameter, EditorRequired]
    public RenderFragment? Content { get; set; } = default!;
    [Parameter, EditorRequired]
    public HtmxAttributes HtmxAttributes { get; set; } = default!;
}
```

The `RegisterProductPage.razor` file using the new components will look like this:

```xml
@using MyApp.RazorComponents;
<h4>Register Product</h4>
<Section>
    <Content>
        <Form HtmxAttributes=@(new HtmxAttributes("/products/register","#main", "innerHTML"))>
            <Content>
                <div class="row mb-4">
                    <div class="col-4">
                        <TextInput Property="Name"
                                   Label="Name"
                                   placeholder="Enter name"
                                   maxlength=100
                                   required />
                    </div>
                    <div class="col-4">
                        <NumericInput Property="Price"
                                      Label="Price"
                                      Prefix="$"
                                      step="0.01"
                                      min="0.01"
                                      value="0"
                                      required />
                    </div>
                    <div class="col-4">
                        <SelectInput Property="Unit"
                                     Label="Unit"
                                     required
                                     Source=@(new Dictionary<string, string>(){{"UN","Unit"},{"KG","Kilogram"}}) />
                    </div>
                </div>
                <div class="row mb-4">
                    <div class="col">
                        <TextAreaInput Property="Description"
                                       Label="Description"
                                       rows="5"
                                       placeholder="Enter description"
                                       maxlength=500 />
                    </div>
                </div>
            </Content>
        </Form>
    </Content>
</Section>
@code {
}
```

## Edit Product

Create a `Products/EditProduct.cs` file with:

```csharp
using Microsoft.AspNetCore.Http.HttpResults;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
namespace MyApp.Products;

public static class EditProduct
{
    public class Request
    {
        public string? Description { get; set; }
        public decimal Price { get; set; }
        public Unit Unit { get; set; }
    }

    public static async Task<RazorComponentResult> HandlePage([FromRoute] Guid productId, [FromServices] MyAppDbContext appDbContext)
    {
        var product = await appDbContext.Set<Product>().AsNoTracking().FirstAsync(p => p.ProductId == productId);
        return new RazorComponentResult<EditProductPage>(new { Product = product });
    }

    public static async Task<RazorComponentResult> HandleAction([FromRoute] Guid productId, [FromServices] MyAppDbContext appDbContext, [FromBody] Request request)
    {
        var product = await appDbContext.Set<Product>().FindAsync(productId);
        product.Description = request.Description;
        product.Price = request.Price;
        product.Unit = request.Unit;
        await appDbContext.SaveChangesAsync();
        return await ListProducts.HandlePage(appDbContext, new ListProducts.Request());
    }
}
```

The `HandlePage` method will receive the request to render the edit form, and the `HandleAction` method will process the request from the form. Update the `Endpoints.cs` file as follows:

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
        group.MapGet("/{productId:guid}/edit", EditProduct.HandlePage);
        group.MapPost("/{productId:guid}/edit", EditProduct.HandleAction);
    }
}
```

Let's create a new component, `RazorComponents.ActionLink.razor`:

```xml
<a class="icon-link"
   href="#"
   hx-get=@HtmxAttributes.Endpoint
   hx-target=@HtmxAttributes.Target
   hx-swap=@HtmxAttributes.Swap>
    <span class=@Icon></span>
</a>
@code {
    [Parameter, EditorRequired]
    public string Icon { get; set; } = default!;
    [Parameter, EditorRequired]
    public HtmxAttributes HtmxAttributes { get; set; } = default!;
}
```

Modify the `Products/ListProductsPage.razor` file by changing the `<DataTable>` element with:

```xml
<DataTable Items=@Results
		   Context="item">
	<TableHeader>
		<th></th>
		<th>#</th>
		<th>Name</th>
		<th>Unit</th>
		<th>Price</th>
		<th>Is Enabled</th>
	</TableHeader>
	<RowTemplate>
		<td>
			<div class="hstack gap-1">
				<ActionLink Icon="bi bi-pencil" 
                            HtmxAttributes=@(new HtmxAttributes($"/products/{item.ProductId}/edit", "#main", "innerHTML")) />
			</div>
		</td>
		<td>@item.ProductId</td>
		<td>@item.Name</td>
		<td>@item.Unit</td>
		<td>@item.Price</td>
		<td>@item.IsEnabled</td>
	</RowTemplate>
</DataTable>
```

The `ActionLink` component will render the `Products/EditProductPage.razor` component detailed below:

```xml
@using MyApp.RazorComponents;
<h4>Edit Product</h4>
<Section>
    <Content>
        <Form HtmxAttributes=@(new HtmxAttributes($"/products/{Product.ProductId}/edit","#main", "innerHTML"))>
            <Content>
                <div class="row mb-4">
                    <div class="col-4">
                        <TextInput Property="Name"
                                   Label="Name"
                                   placeholder="Enter name"
                                   maxlength=100
                                   value=@Product.Name
                                   disabled
                                   readonly />
                    </div>
                    <div class="col-4">
                        <NumericInput Property="Price"
                                      Label="Price"
                                      Prefix="$"
                                      step="0.01"
                                      min="0.01"
                                      value=@Product.Price
                                      required />
                    </div>
                    <div class="col-4">
                        <SelectInput Property="Unit"
                                     Label="Unit"
                                     required
                                     Value=@Product.Unit.ToString()
                                     Source=@(new Dictionary<string, string>(){{"UN","Unit"},{"KG","Kilogram"}}) />
                    </div>
                </div>
                <div class="row mb-4">
                    <div class="col">
                        <TextAreaInput Property="Description"
                                       Label="Description"
                                       rows="5"
                                       value=@Product.Description
                                       placeholder="Enter description"
                                       maxlength=500 />
                    </div>
                </div>
            </Content>
        </Form>
    </Content>
</Section>
@code {
    [Parameter, EditorRequired]
    public Product Product { get; set; } = default!;
}
```

The main difference with the `RegisterProductPage.razor` component is that we have a `Product` parameter to set the current values in the form. Additionally, the `Name` field is not editable. Run the application:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1717250584328/27ba862b-8ac0-4ce3-8304-9441cb3906c3.png align="center")

You can find the final code [here](https://github.com/raulnq/htmx-dotnet-app/tree/part-ii). Thank you, and happy coding.