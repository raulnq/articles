---
title: "Developing Your First App with HTMX and .NET: Part VI"
datePublished: Tue Jul 30 2024 00:47:59 GMT+0000 (Coordinated Universal Time)
cuid: clz7p8izb000308jnfqfub2h2
slug: developing-your-first-app-with-htmx-and-net-part-vi
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1722259711789/b630f745-57b6-4027-8a11-049edaeca950.png
tags: net, dotnet, htmx

---

In our previous articles, we used inline forms to register or edit products directly within the page's main content. In this part, we will explore modal forms, which appear in a pop-up overlaying the page's main content, typically dimming or blurring the background. You can download the starting code from the previous [article](https://blog.raulnq.com/developing-your-first-app-with-htmx-and-net-part-v).

To see the modal forms in action, we will implement a soft delete feature for our products, which prompts for a reason during the process. The first step is to define a placeholder for the modal form in the `MainPage.razor` file. Copy the following lines to the end of the `<body>` tag:

```xml
<div id="modal" class="modal fade">
    <div id="modal-dialog" class="modal-dialog"></div>
</div>
```

We are using the Bootstrap [**modal**](https://getbootstrap.com/docs/5.3/components/modal/#how-it-works) component to create our modal form. The next step is to create a way to invoke our modal form. Let's create a `RazorComponents/ShowModal.razor` file with the following content:

```xml
<li>
    <a class="dropdown-item"
       href="#"
       hx-get=@HtmxAttributes.Endpoint
       hx-target=@HtmxAttributes.Target
       hx-swap=@HtmxAttributes.Swap>@Text</a>
</li>

@code {
    [Parameter, EditorRequired]
    public string Text { get; set; } = default!;
    [Parameter, EditorRequired]
    public HtmxAttributes HtmxAttributes { get; set; } = default!;
}
```

Add the new component inside the `<MenuItems>` tag in the `Products/EditProductPage.razor` file to invoke the endpoint that returns the HTML content of our modal form:

```xml
<MenuItems>
	<MenuItem Text="Enable"
			  IsDisabled=@(Product.IsEnabled)
			  HtmxAttributes=@(new HtmxAttributes($"/products/{Product.ProductId}/enable", "#main", "innerHTML"){Confirm="Are you sure?"}) />
	<MenuItem Text="Disable"
			  IsDisabled=@(!Product.IsEnabled)
			  Confirm="Are you sure?"
			  HtmxAttributes=@(new HtmxAttributes($"/products/{Product.ProductId}/disable", "#main", "innerHTML")) />
	<ShowModal Text="Delete"
			   HtmxAttributes=@(new HtmxAttributes($"/products/{Product.ProductId}/delete", "#modal-dialog", "innerHTML")) />
</MenuItems>
```

Something to note here is that we are using the `id` of the new `div` defined in the `MainPage.razor` file as the HTMX target. Let's continue creating the `RazorComponents/ModalForm.razor` file as follows:

```xml
<div class="modal-content">
    <form hx-post=@HtmxAttributes.Endpoint
          hx-target=@HtmxAttributes.Target
          hx-swap=@HtmxAttributes.Swap
          hx-ext="json-enc"
          hx-indicator="#modal-form-spinner"
          hx-disabled-elt="#modal-form-button"
          hx-select=@HtmxAttributes.Select>
        <div class="modal-header">
            <h1 class="modal-title">@Title</h1>
            <button type="button" class="btn-close" data-bs-dismiss="modal"></button>
        </div>
        <div class="modal-body">
            @Content
        </div>
        <div class="modal-footer">
            <button type="button" class="btn btn-secondary" data-bs-dismiss="modal">Close</button>
            <button id="modal-form-button" type="submit" class="btn btn-primary">
                <span>Save Changes</span>
                <span id="modal-form-spinner" class="spinner-border spinner-border-sm htmx-indicator" aria-hidden="true" />
            </button>
        </div>
    </form>
</div>


@code {
    [Parameter, EditorRequired]
    public RenderFragment? Content { get; set; } = default!;
    [Parameter, EditorRequired]
    public string Title { get; set; } = default!;
    [Parameter, EditorRequired]
    public HtmxAttributes HtmxAttributes { get; set; } = default!;
}
```

This component is similar to `RazorComponents/Form.razor`, but uses the Bootstrap modal component. Based on this component, we will create the `RazorComponents/DeleteProductPage.razor` file as follows:

```xml
@using MyApp.RazorComponents;
<ModalForm HtmxAttributes=@(new HtmxAttributes($"/products/{ProductId}/delete", "#main"))
           Title="">
    <Content>
        <div class="row mb-4">
            <div class="col">
                <TextInput Property=@nameof(DeleteProduct.Request.Reason)
                           Label="Reason"
                           maxlength=500
                           placeholder="Enter reason"
                           required />
            </div>
        </div>
    </Content>
</ModalForm>

@code {
    [Parameter, EditorRequired]
    public Guid ProductId { get; set; } = default;
}
```

The HTMX target will be the main content div. Now, it's time to implement the endpoints to render and process the action. Create the `Products/DeleteProduct.cs` file with the following content:

```csharp
using Microsoft.AspNetCore.Http.HttpResults;
using Microsoft.AspNetCore.Mvc;
namespace MyApp.Products;

public static class DeleteProduct
{
    public class Request
    {
        public string? Reason { get; set; }
    }

    public static Task<RazorComponentResult> HandlePage(
        [FromRoute] Guid productId,
        HttpContext context)
    {
        context.Response.Headers.Append("HX-Trigger-After-Swap", @"{""openModalEvent"":""true""}");
        return Task.FromResult<RazorComponentResult>(new RazorComponentResult<DeleteProductPage>(new
        {
            ProductId = productId,
        }));
    }

    public static async Task<RazorComponentResult> HandleAction(
        [FromRoute] Guid productId,
        [FromServices] MyAppDbContext appDbContext,
        [FromBody] Request request,
        HttpContext httpContext)
    {
        var product = await appDbContext.Set<Product>().FindAsync(productId);
        product.DeletedAt = DateTimeOffset.Now;
        product.DeletionReason = request.Reason;
        await appDbContext.SaveChangesAsync();

        httpContext.Response.Headers.Append("HX-Trigger-After-Swap", @$"{{""successEvent"":""The product was deleted successfully"", ""closeModalEvent"":""true""}}");
        return await ListProducts.HandlePage(appDbContext, new ListProducts.Request(), httpContext);
    }
}
```

The `HandlePage` method will return the HTML of the modal form and an [`HX-Trigger`](https://htmx.org/headers/hx-trigger/) response header to trigger an `openModalEvent` custom event as soon as the client receives the response (we already explained this HTMX feature [here](https://blog.raulnq.com/developing-your-first-app-with-htmx-and-net-part-iii)). The `HandleAction` method will soft delete the product and return a `successEvent` and `closeModalEvent` custom event. The HTML returned will be the updated list of products. Add the following script in the `MainPage.razor` file:

```javascript
<script>
	const modal = bootstrap.Modal.getOrCreateInstance(document.getElementById("modal"))
	htmx.on("openModalEvent", (e) => {
		modal.show()
	})
	htmx.on("closeModalEvent", (e) => {
		modal.hide()
	})
	htmx.on("hidden.bs.modal", () => {
		document.getElementById("modal-dialog").innerHTML = ""
	})
</script>
```

In this section, we add handlers for the HTMX custom events to show and close the form modal. We also add a handler for the `hidden.bs.modal` event, which fires when the modal is hidden, to clean up the content. Add the following line to the `Products/Endpoints.cs` file:

```csharp
group.MapGet("/{productId:guid}/delete", DeleteProduct.HandlePage);
group.MapPost("/{productId:guid}/delete", DeleteProduct.HandleAction);
```

As a final step, let's update the `HandlePage` method in the `Product/ListProducts.cs` file as follows:

```csharp
public static async Task<RazorComponentResult> HandlePage(
	[FromServices] MyAppDbContext appDbContext,
	[AsParameters] Request request,
	HttpContext httpContext,
	[FromQuery] int page = 1,
	[FromQuery] int pageSize = 10)
{
	var name = request.Name ?? string.Empty;
	var results = await appDbContext.Set<Product>()
		.AsNoTracking().Where(p => p.Name.Contains(name) && p.DeletedAt == null).ToPagedListAsync(page, pageSize);
	var uri = "/products/list".AddQuery(new KeyValuePair<string, StringValues>[]
	{
		new("Name", request.Name),
		new("Page", page.ToString()),
		new("PageSize", pageSize.ToString())
	});
	httpContext.Response.Headers.Append("HX-Push-Url", uri.PathAndQuery);
	return new RazorComponentResult<ListProductsPage>(new { Results = results, Query = uri.Query });
}
```

Run the application and test the new soft delete feature:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1722297330969/47dc7f62-d0af-4ca0-a86c-9d54d148818f.png align="center")

You can find the final code [here](https://github.com/raulnq/htmx-dotnet-app/tree/part-vi). Thank you, and happy coding.