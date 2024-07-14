---
title: "Developing Your First App with HTMX and .NET: Part V"
datePublished: Sun Jul 14 2024 01:45:59 GMT+0000 (Coordinated Universal Time)
cuid: clykw9hi9000009li1mwc68tw
slug: developing-your-first-app-with-htmx-and-net-part-v
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1720800815117/9cf42df3-e8b2-44ed-9b89-cd9e536fbe47.png
tags: net, dotnet, htmx

---

In many applications, confirmation dialogs are essential for specific actions to ensure users are fully aware of the changes they are making. In our scenario, we will implement two actions that require user confirmation: one to enable a product and another to disable it, as shown in the following image:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1720889532407/d13f0813-08e7-48e2-855c-c9ced5c22966.png align="center")

You can download the starting code from the previous [article](https://blog.raulnq.com/developing-your-first-app-with-htmx-and-net-part-iv). Open the solution and add the `RazorComponents/Breadcrumbs.razor` file with the following content:

```csharp
<nav class="d-flex justify-content-between align-middle">
    <ol class="breadcrumb">
        @foreach (var link in Links)
        {
            if (link.IsActive)
            {
                <li class="breadcrumb-item active">@link.Name</li>
            }
            else
            {
                <li class="breadcrumb-item">
                    <a href="#"
                       hx-get=@link.HtmxAttributes!.Endpoint
                       hx-target=@link.HtmxAttributes!.Target
                       hx-swap=@link.HtmxAttributes!.Swap>@link.Name</a>
                </li>
            }
        }
    </ol>
    @if (MenuItems != null)
    {
        <div class="dropdown d-inline">
            <button class="btn btn-secondary" type="button" data-bs-toggle="dropdown" aria-expanded="false">
                <i class="bi bi-three-dots-vertical"></i>
            </button>
            <ul class="dropdown-menu">
                @MenuItems
            </ul>
        </div>
    }
</nav>

@code {
    [Parameter]
    public RenderFragment? MenuItems { get; set; } = default!;
    [Parameter, EditorRequired]
    public IEnumerable<Link> Links { get; set; } = default!;

    public class Link
    {
        public HtmxAttributes? HtmxAttributes { get; set; }
        public string? Name { get; set; }
        public bool IsActive { get; set; }
        public Link(string name)
        {
            Name = name;
            IsActive = true;
        }
        public Link(string name, HtmxAttributes htmxAttributes)
        {
            Name = name;
            HtmxAttributes = htmxAttributes;
            IsActive = false;
        }
    }
}
```

To create this new Razor component, we use the [breadcrumb](https://getbootstrap.com/docs/5.3/components/breadcrumb/) and [dropdown](https://getbootstrap.com/docs/5.3/components/dropdowns/) Bootstrap components. Let's open the `RazorComponents/HtmxAttributes.cs` file and update the content as follows:

```csharp
public class HtmxAttributes
{
    public string Target { get; set; } = default!;
    public string Swap { get; set; } = "innerHTML";
    public string Endpoint { get; set; } = default!;
    public string Select { get; set; } = default!;
    public string Confirm { get; set; } = default!;
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

We added the `Confirm` property to support the [`hx-confirm`](https://htmx.org/attributes/hx-confirm/) HTMX attribute, which provides a simple way to confirm a user action before proceeding. Add the `RazorComponents/MenuItem.razor` file with the following content:

```csharp
<li>
	<a class="dropdown-item @(IsDisabled ? "disabled" :"")"
	   href="#"
	   hx-post=@HtmxAttributes.Endpoint
	   hx-target=@HtmxAttributes.Target
	   hx-confirm=@HtmxAttributes.Confirm
	   hx-swap=@HtmxAttributes.Swap>@Text</a>
</li>

@code {
    [Parameter, EditorRequired]
    public string Text { get; set; } = default!;
    [Parameter, EditorRequired]
    public HtmxAttributes HtmxAttributes { get; set; } = default!;
    [Parameter]
    public bool IsDisabled { get; set; } = false;
}
```

The component above represents each dropdown item used in the first component. Add the `RazorComponents/StatusBadge.razor` file with the following content:

```csharp
@if (IsEnabled)
{
    <h5><span class="badge text-bg-success">Enable</span></h5>
}
else
{
    <h5><span class="badge text-bg-warning">Disable</span></h5>
}
@code {

    [Parameter, EditorRequired]
    public bool IsEnabled { get; set; } = default!;
}
```

We will use this component to show the current status of the product. The component was built using the [badge](https://getbootstrap.com/docs/5.3/components/badge/) Bootstrap component. Now, let's use our new components. Open the `EditProductPage.razor` file and add the following code snippet after the `h4` tag:

```xml
<Breadcrumbs Links=@(new []{
             new Breadcrumbs.Link("List products", new HtmxAttributes("/products/list", "#main", "innerHTML")),
             new Breadcrumbs.Link("Edit product")})>
    <MenuItems>
        <MenuItem Text="Enable"
                  IsDisabled=@(Product.IsEnabled)
                  HtmxAttributes=@(new HtmxAttributes($"/products/{Product.ProductId}/enable", "#main", "innerHTML"){Confirm="Are you sure?"}) />
        <MenuItem Text="Disable"
                  IsDisabled=@(!Product.IsEnabled)
                  HtmxAttributes=@(new HtmxAttributes($"/products/{Product.ProductId}/disable", "#main", "innerHTML"){Confirm="Are you sure?"}) />
    </MenuItems>
</Breadcrumbs>
<nav class="navbar hstack gap-3 justify-content-end">
    <StatusBadge IsEnabled=@Product.IsEnabled />
</nav>
```

Add a similar code snippet to the `ListProductsPage.razor` file:

```xml
<Breadcrumbs Links=@(new []{new Breadcrumbs.Link("List products")}) />
```

And in the `ListProductsPage.razor` file:

```xml
<Breadcrumbs Links=@(new []{
             new Breadcrumbs.Link("List products", new HtmxAttributes("/products/list", "#main", "innerHTML")),
             new Breadcrumbs.Link("Register product")}) />
```

Add the `Products/EnableProduct.cs` file with the following content:

```csharp
using Microsoft.AspNetCore.Http.HttpResults;
using Microsoft.AspNetCore.Mvc;
namespace MyApp.Products;

public static class EnableProduct
{
    public static async Task<RazorComponentResult> HandleAction(
        [FromRoute] Guid productId,
        [FromServices] MyAppDbContext appDbContext,
        HttpContext httpContext)
    {
        var product = await appDbContext.Set<Product>().FindAsync(productId);
        product.IsEnabled = true;
        await appDbContext.SaveChangesAsync();
        httpContext.Response.Headers.Append("HX-Trigger-After-Swap", @$"{{""successEvent"":""The product was enabled successfully""}}");
        return await EditProduct.HandlePage(productId, appDbContext);
    }
}
```

And the `Products/DisableProduct.cs` file with the following content:

```csharp
using Microsoft.AspNetCore.Http.HttpResults;
using Microsoft.AspNetCore.Mvc;
namespace MyApp.Products;

public static class DisableProduct
{
    public static async Task<RazorComponentResult> HandleAction(
        [FromRoute] Guid productId,
        [FromServices] MyAppDbContext appDbContext,
        HttpContext httpContext)
    {
        var product = await appDbContext.Set<Product>().FindAsync(productId);
        product.IsEnabled = false;
        await appDbContext.SaveChangesAsync();
        httpContext.Response.Headers.Append("HX-Trigger-After-Swap", @$"{{""successEvent"":""The product was disabled successfully""}}");
        return await EditProduct.HandlePage(productId, appDbContext);
    }
}
```

These methods will manage the new features. Both methods will return the same HTML generated by the edit endpoint. In the `Products/Endpoints.cs` file, add the following lines:

```csharp
group.MapPost("/{productId:guid}/disable", DisableProduct.HandleAction);
group.MapPost("/{productId:guid}/enable", EnableProduct.HandleAction);
```

Now that everything is set up, let's test our new actions:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1720895473614/4eedee07-ef8f-4b89-b5f7-2ad8891a2729.png align="center")

As we can see, the current implementation uses the native browser confirm dialog. However, with a bit more effort, we could create something more advanced. To do this, we will use the [sweetalert2](https://sweetalert2.github.io/) library. Open the `MainPage.razor` file and add the following code:

```javascript
<script src="https://cdn.jsdelivr.net/npm/sweetalert2@11"></script>
<script>
	function showMessage(event, message, eventToTrigger) {
		Swal.fire({
			title: message,
			customClass: {
				title: 'text-center h2 mt-3 mb-3 text-body',
				confirmButton: 'btn btn-primary',
				cancelButton: 'btn btn-secondary',
				closeButton: 'btn-close',
			},
			showCancelButton: true,
			confirmButtonText: 'Yes',
			cancelButtonText: 'No',
		}).then((result) => {
			if (result.isConfirmed) {
				htmx.trigger(event.target, eventToTrigger)
			}
		})
	}
</script>
```

In this code snippet, we are adding a reference to the library via CDN and defining a method to show the confirmation dialog. The `showMessage` function will display a modal to confirm an action. If confirmed, it will [trigger](https://htmx.org/api/#trigger) a custom event on an element. Note that we are styling the modal with some Bootstrap classes to maintain the same look and feel. Open the `RazorComponents/MenuItem.razor` file and update the content as follows:

```csharp
@if (string.IsNullOrEmpty(Confirm))
{
    <li>
        <a class="dropdown-item @(IsDisabled ? "disabled" :"")"
           href="#"
           hx-post=@HtmxAttributes.Endpoint
           hx-target=@HtmxAttributes.Target
           hx-confirm=@HtmxAttributes.Confirm
           hx-swap=@HtmxAttributes.Swap>@Text</a>
    </li>
}
else
{
    <li>
        <a class="dropdown-item @(IsDisabled ? "disabled" :"")"
           href="#"
           hx-post=@HtmxAttributes.Endpoint
           hx-target=@HtmxAttributes.Target
           hx-confirm=@HtmxAttributes.Confirm
           hx-swap=@HtmxAttributes.Swap
           hx-trigger="confirmed"
           hx-on:click="showConfirm(event,'@Confirm','confirmed')">@Text</a>
    </li>
}


@code {
    [Parameter, EditorRequired]
    public string Text { get; set; } = default!;
    [Parameter, EditorRequired]
    public HtmxAttributes HtmxAttributes { get; set; } = default!;
    [Parameter]
    public bool IsDisabled { get; set; } = false;
    [Parameter]
    public string Confirm { get; set; } = default!;
}
```

We are adding the `Confirm` parameter to the previous component. When it is set, we will render different HTML that uses the [`hx-on`](https://htmx.org/attributes/hx-on/) attribute. This attribute lets us embed scripts inline to respond to events directly on the element. In this case, we want to respond to the `click` event, so we use the `hx-on:click` attribute. To complete the setup, we use the [`hx-trigger`](https://htmx.org/attributes/hx-trigger/) attribute to trigger the request when a `confirmed` event occurs. Open the `EditProductPage.razor` file and replace the menu item related to the disable action with the following content:

```xml
<MenuItem Text="Disable"
		  IsDisabled=@(!Product.IsEnabled)
		  Confirm="Are you sure?"
		  HtmxAttributes=@(new HtmxAttributes($"/products/{Product.ProductId}/disable", "#main", "innerHTML")) />
```

Run the application and test the new confirmation dialog:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1720918552423/d18163c5-390b-451e-a03e-bd8679dc9a4c.png align="center")

You can find the final code [here](https://github.com/raulnq/htmx-dotnet-app/tree/part-v). Thank you, and happy coding.