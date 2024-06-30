---
title: "Developing Your First App with HTMX and .NET: Part IV"
datePublished: Sun Jun 30 2024 02:13:11 GMT+0000 (Coordinated Universal Time)
cuid: cly0x2jhr000109l9btlr2t6f
slug: developing-your-first-app-with-htmx-and-net-part-iv
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1719639554459/6cb66419-a03c-424f-b0a7-3a1660a5933e.png
tags: net, dotnet, htmx

---

Continuing with our application development, we will now implement pagination for our product list page, as shown in the following image:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1719641049120/c615d008-8845-4ec3-9a2f-b4a5d9d3c243.png align="center")

You can download the starting code from the previous [article](https://blog.raulnq.com/developing-your-first-app-with-htmx-and-net-part-iii). Open the solution and add a `ListResponse.cs` file with the following content:

```csharp
namespace MyApp;

public class ListResponse<T>
{
    public int Page { get; set; }
    public int PageSize { get; set; }
    public int TotalCount { get; set; }
    public IEnumerable<T> Items { get; set; }
    public ListResponse(IEnumerable<T> items, int page, int pageSize, int totalCount)
    {
        Page = page;
        PageSize = pageSize;
        TotalCount = totalCount;
        Items = items;
    }
}
```

The class above wraps our query results with their pagination information. Add the `Extension.cs` file as follows:

```csharp
using Microsoft.AspNetCore.Http.Extensions;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Primitives;
namespace MyApp;

public static class Extensions
{
    public static async Task<ListResponse<T>> ToPagedListAsync<T>(
    this IQueryable<T> source,
    int page,
    int pageSize)
    {
        var totalCount = await source.CountAsync();
        if (totalCount > 0)
        {
            var items = await source
                .Skip((page - 1) * pageSize)
                .Take(pageSize)
                .ToListAsync();
            return new ListResponse<T>(items, page, pageSize, totalCount);
        }
        return new(Enumerable.Empty<T>(), 0, 0, 0);
    }

    public static Uri AddQuery(this string path, IEnumerable<KeyValuePair<string, StringValues>> query)
    {
        var queryBuilder = new QueryBuilder(query);
        var uriBuilder = new UriBuilder()
        {
            Path = path,
            Query = queryBuilder.ToString()
        };
        return uriBuilder.Uri;
    }
}
```

The `ToPagedListAsync` extension method is used to retrieve and wrap the paged data. The `AddQuery` method helps us build a URI based on a path and the query parameters. Update the `Products/ListProducts.cs` file as follows:

```csharp
using Microsoft.AspNetCore.Http.HttpResults;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Primitives;
namespace MyApp.Products;
public static class ListProducts
{
    public class Request
    {
        public string? Name { get; set; }
    }

    public static async Task<RazorComponentResult> HandlePage(
        [FromServices] MyAppDbContext appDbContext,
        [AsParameters] Request request,
        HttpContext httpContext,
        [FromQuery] int page = 1,
        [FromQuery] int pageSize = 10)
    {
        var name = request.Name ?? string.Empty;
        var results = await appDbContext.Set<Product>()
            .AsNoTracking().Where(p => p.Name.Contains(name)).ToPagedListAsync(page, pageSize);
        var uri = "/products/list".AddQuery(new KeyValuePair<string, StringValues>[]
        {
            new("Name", request.Name),
            new("Page", page.ToString()),
            new("PageSize", pageSize.ToString())
        });
        httpContext.Response.Headers.Append("HX-Push-Url", uri.PathAndQuery);
        return new RazorComponentResult<ListProductsPage>(new { Results = results, Query = uri.Query });
    }
}
```

We are adding two parameters to the `HandlePage` method: one to specify the current page and another to select the page size. To retrieve the data from the database, we are using the `ToPagedListAsync` extension method previously created. The last change involves building the corresponding URI for our endpoint. Add the `RazorComponents/Pagination.razor` file with the following content:

```csharp
@using Microsoft.AspNetCore.Http.Extensions
@using Microsoft.Extensions.Primitives
@using System.Web

<nav class="btn-toolbar">
    <ul class="pagination">
        <li class="page-item @(_hasPreviousPage?"":"disabled")">
            <a class="page-link"
               href="#"
               hx-get=@(_hasPreviousPage?GetUri(Page-1, PageSize):string.Empty)
               hx-swap=@HtmxAttributes.Swap
               hx-target=@HtmxAttributes.Target
               hx-select=@HtmxAttributes.Select>Previous</a>

        </li>
        @for (int i = 1; i <= _totalPages; i++)
        {
            <li class="page-item @(Page==i?"active":"")">
                <a class="page-link"
                   href="#"
                   hx-get=@GetUri(i, PageSize)
                   hx-swap=@HtmxAttributes.Swap
                   hx-target=@HtmxAttributes.Target
                   hx-select=@HtmxAttributes.Select>@i</a>
            </li>

        }
        <li class="page-item @(_hasNextPage?"":"disabled")">
            <a class="page-link"
               href="#"
               hx-get=@(_hasNextPage?GetUri(Page+1, PageSize):string.Empty)
               hx-swap=@HtmxAttributes.Swap
               hx-target=@HtmxAttributes.Target
               hx-select=@HtmxAttributes.Select>Next</a>
        </li>
    </ul>
    <div class="dropdown ms-auto">
        <button class="btn btn-primary dropdown-toggle" type="button" data-bs-toggle="dropdown">
            Rows per page (@PageSize)
        </button>
        <ul class="dropdown-menu">
            <li>
                <a class="dropdown-item"
                   hx-get="@GetUri(1, 10)"
                   hx-swap=@HtmxAttributes.Swap
                   hx-target=@HtmxAttributes.Target
                   hx-select=@HtmxAttributes.Select
                   href="#">10</a>
            </li>
            <li>
                <a class="dropdown-item"
                   hx-get="@GetUri(1, 20)"
                   hx-swap=@HtmxAttributes.Swap
                   hx-target=@HtmxAttributes.Target
                   hx-select=@HtmxAttributes.Select
                   href="#">20</a>
            </li>
            <li>
                <a class="dropdown-item"
                   hx-get="@GetUri(1, 40)"
                   hx-swap=@HtmxAttributes.Swap
                   hx-target=@HtmxAttributes.Target
                   hx-select=@HtmxAttributes.Select
                   href="#">40</a>
            </li>
        </ul>
    </div>
</nav>
@code {
    [Parameter, EditorRequired]
    public int Page { get; set; } = 1;
    [Parameter, EditorRequired]
    public int PageSize { get; set; } = 10;
    [Parameter, EditorRequired]
    public int TotalCount { get; set; } = default!;
    [Parameter, EditorRequired]
    public HtmxAttributes HtmxAttributes { get; set; } = default!;
    [Parameter, EditorRequired]
    public string Query { get; set; } = default!;
    private int _totalPages;
    private bool _hasPreviousPage;
    private bool _hasNextPage;

    protected override void OnParametersSet()
    {
        _totalPages = (int)Math.Ceiling(TotalCount / (double)PageSize);
        _hasPreviousPage = Page > 1;
        _hasNextPage = Page < _totalPages;
    }

    public string? GetUri(int page, int pageSize)
    {
        var query = HttpUtility.ParseQueryString(Query);
        query["Page"] = page.ToString();
        query["PageSize"] = pageSize.ToString();
        var uriBuilder = new UriBuilder()
            {
                Path = HtmxAttributes.Endpoint,
                Query = query.ToString()
            };
        return uriBuilder.Uri.PathAndQuery;
    }
}
```

We use two Bootstrap components to build this Razor component: [pagination](https://getbootstrap.com/docs/5.3/components/pagination/) and [dropdown](https://getbootstrap.com/docs/5.3/components/dropdowns/). The component's inputs are the current page, the page size, the total number of records, the HTMX attributes, and the query parameters. The [`OnParametersSet`](https://learn.microsoft.com/en-us/aspnet/core/blazor/components/lifecycle?view=aspnetcore-8.0#after-parameters-are-set-onparameterssetasync) method calculates the total number of pages and whether to enable the previous and next page buttons. The method `GetUri` is used to build the correct URLs. Run the application to see the new Razor component in action:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1719712107452/809fe3db-e96b-4532-aae1-101478ee6b75.png align="center")

You can find the final code [here](https://github.com/raulnq/htmx-dotnet-app/tree/part-iv). Thank you, and happy coding.