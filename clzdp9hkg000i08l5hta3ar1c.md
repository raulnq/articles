---
title: "Developing Your First App with HTMX and .NET: Part VII"
datePublished: Sat Aug 03 2024 05:35:21 GMT+0000 (Coordinated Universal Time)
cuid: clzdp9hkg000i08l5hta3ar1c
slug: developing-your-first-app-with-htmx-and-net-part-vii
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1722646989437/f969f62e-e921-4532-8a03-0845a8c9c9d0.png
tags: net, dotnet, htmx

---

So far, when an exception occurs in the application, a toast notification displays the details. Taking advantage of this mechanism, we throw exceptions when certain business rules are unmet.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1722648161050/2a4cb695-9439-4f3c-bbdc-7ffc79c0f8e0.avif align="center")

While this approach is practical, we can enhance it by showing the message directly within the form. Let's start by updating the `RazorComponents/TextInput.razor` as follows:

```csharp
<div class="form-group">
    <label for=@Property class="form-label">@Label</label>
    <input type="text"
           class="form-control @(!string.IsNullOrEmpty(ErrorMessage)?"is-invalid":"")"
           id=@Property
           name=@Property
           @attributes="Attributes" />
    @if (!string.IsNullOrEmpty(ErrorMessage))
    {
        <div id=@($"{Property}-error-message") class="invalid-feedback">
            @ErrorMessage
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
    public string ErrorMessage { get; set; } = default!;
}
```

By setting the `ErrorMessage` property, a message will be displayed along with the input. Create a `Products/ProductsForm.razor` file with the following content:

```csharp
@using MyApp.RazorComponents;
<div id="product-form">
    <div class="row mb-4">
        <div class="col-4">
            @if (InEdition)
            {
                <TextInput Property="Name"
                           Label="Name"
                           placeholder="Enter name"
                           maxlength=100
                           value=@Name
                           disabled
                           readonly />
            }
            else
            {
                <TextInput Property="Name"
                           Label="Name"
                           placeholder="Enter name"
                           maxlength=100
                           value=@Name
                           ErrorMessage=@NameErrorMessage
                           required />
            }

        </div>
        <div class="col-4">
            <NumericInput Property="Price"
                          Label="Price"
                          Prefix="$"
                          step="0.01"
                          min="0.01"
                          value=@Price
                          required />
        </div>
        <div class="col-4">
            <SelectInput Property="Unit"
                         Label="Unit"
                         Value=@Unit
                         required
                         Source=@(new Dictionary<string, string>(){{"UN","Unit"},{"KG","Kilogram"}}) />
        </div>
    </div>
    <div class="row mb-4">
        <div class="col">
            <TextAreaInput Property="Description"
                           Label="Description"
                           rows="5"
                           value=@Description
                           placeholder="Enter description"
                           maxlength=500 />
        </div>
    </div>
</div>
@code {
    [Parameter]
    public string Name { get; set; } = default!;
    [Parameter]
    public string NameErrorMessage { get; set; } = default!;
    [Parameter]
    public decimal Price { get; set; } = 0;
    [Parameter]
    public string Unit { get; set; } = default!;
    [Parameter]
    public string Description { get; set; } = default!;
    [Parameter]
    public bool InEdition { get; set; } = false;
}
```

This file is the form content of the `Products/RegisterProductPage.razor` and `Products/EditProductPage.razor` files. Update the `Products/RegisterProductPage.razor` file to use the new component:

```xml
@using MyApp.RazorComponents;
<h4>Register Product</h4>
<Breadcrumbs Links=@(new []{
             new Breadcrumbs.Link("List products", new HtmxAttributes("/products/list", "#main", "innerHTML")),
             new Breadcrumbs.Link("Register product")}) />
<Section>
    <Content>
        <Form HtmxAttributes=@(new HtmxAttributes("/products/register","#main", "innerHTML"))>
            <Content>
                <ProductForm />
            </Content>
        </Form>
    </Content>
</Section>
@code {
}
```

And the `Products/EditProductPage.razor` file:

```xml
@using MyApp.RazorComponents;
<h4>Edit Product</h4>
<Breadcrumbs Links=@(new []{
             new Breadcrumbs.Link("List products", new HtmxAttributes("/products/list", "#main", "innerHTML")),
             new Breadcrumbs.Link("Edit product")})>
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
</Breadcrumbs>
<nav class="navbar hstack gap-3 justify-content-end">
    <StatusBadge IsEnabled=@Product.IsEnabled />
</nav>
<Section>
    <Content>
        <Form HtmxAttributes=@(new HtmxAttributes($"/products/{Product.ProductId}/edit","#main", "innerHTML"))>
            <Content>
                <ProductForm InEdition=true
                             Name=@Product.Name
                             Price=@Product.Price
                             Unit=@Product.Unit.ToString()
                             Description=@Product.Description />
            </Content>
        </Form>
    </Content>
</Section>
@code {
    [Parameter, EditorRequired]
    public Product Product { get; set; } = default!;
}
```

At this point, if we run the application, it will behave the same as it did before all the changes. Update the `HandleAction` method in the `Products/RegisterProduct.cs` file as follows:

```csharp
public static async Task<RazorComponentResult> HandleAction(
	[FromServices] MyAppDbContext appDbContext,
	[FromBody] Request request,
	HttpContext httpContext)
{
	var any = await appDbContext.Set<Product>().AsNoTracking().AnyAsync(p => p.Name == request.Name);
	if (any)
	{
		httpContext.Response.Headers.Append("HX-Reswap", $"outerHTML");
		httpContext.Response.Headers.Append("HX-Retarget", $"#product-form");
		return new RazorComponentResult<ProductForm>(new
		{
			Name = request.Name,
			NameErrorMessage = "The product already exists",
			Price = request.Price,
			Unit = request.Unit.ToString(),
			Description = request.Description
		});
	}
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
	httpContext.Response.Headers.Append("HX-Trigger-After-Swap", @$"{{""successEvent"":""The product was register successfully""}}");
	return await ListProducts.HandlePage(appDbContext, new ListProducts.Request(), httpContext);
}
```

The original exception throwing is replaced with a response that contains the HTML produced by our new component. To override the original `hx-swap` and `hx-target` attributes in the form, we use the `HX-Reswap` and `HX-Retarget` response headers. Run the application and try to create a duplicate product:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1722662055295/718f2613-141d-4318-bc8d-5a3952171311.png align="center")

You can find the final code [here](https://github.com/raulnq/htmx-dotnet-app/tree/part-vii). Thank you, and happy coding.