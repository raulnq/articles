---
title: "Implementing Authorization in .NET APIs Using Azure Entra ID"
datePublished: Wed Mar 05 2025 17:16:05 GMT+0000 (Coordinated Universal Time)
cuid: cm7w6ixtk000009ld3vi1hqi8
slug: implementing-authorization-in-net-apis-using-azure-entra-id
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1741121799386/d4af13d1-66ee-4e30-9041-d4ef1d878255.png
tags: azure, authorization, dotnet, roles, entra-id

---

Authorization is a critical aspect of securing APIs, ensuring that only authenticated users with the right permissions can access specific resources. Unauthorized access to APIs can lead to significant issues, such as data breaches, unauthorized changes, and compliance violations. Azure Entra ID provides app roles, simplifying role-based access control (RBAC) or attribute-based access control (ABAC) for our applications.

App roles allow applications to define specific roles that can be assigned to users, groups, or other applications. These roles are embedded within the access token, allowing for fine-grained authorization within an application.

Typically, applications use groups for authorization, but this approach has several drawbacks:

* The `groups` claim includes the object ID instead of its name.
    
* By default, if a user belongs to more than 200 groups (the limit for JWT tokens), the `groups` claim is omitted from the token. Instead, `_claim_names` and `_claim_sources` claims are included, indicating that the groups must be queried from the Microsoft Graph API.
    
* Groups are broad and shared across multiple applications, complicating the management of fine-grained application authorization.
    

In contrast, app roles address these issues:

* The user's roles are included in the `roles` claim, with the content being the role's name.
    
* App roles are specific to each application, ensuring better granularity.
    
* There is no strict limit on the number of app roles that can be included in the token, although the token size cannot exceed 4 KB.
    

To see app roles in action, follow the steps in the article [Securing .NET API with Microsoft Entra ID](https://blog.raulnq.com/securing-net-api-with-microsoft-entra-id) and then proceed with the steps listed below.

## Add App Roles

Open our **App Registration** and go to the **App Roles** option. Click **Create app role:**

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1741134132759/a6ec1dcc-ec8d-4acf-97a2-d5bbbf71573a.png align="center")

The **allowed member type** property specifies whether the app role can be assigned to users/groups, applications, or both. The **value** property specifies the content included in the `roles` claim.

## Assign the App Role to a User

Go to **Enterprise Applications** and choose the application where the app role was created. Go to **Users and Groups** and Click **Add User**. Choose the user and select the **Reader** role:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1741134635909/609114a2-cacf-4aa9-b5e4-452806bd1267.png align="center")

## API

Go to the `Program.cs` file and update the content as follows:

```csharp
using Microsoft.AspNetCore.Authentication.JwtBearer;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();
builder.Services.AddCors();
builder.Logging.SetMinimumLevel(LogLevel.Debug);

builder.Services.AddAuthentication(options =>
{
    options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
})
.AddJwtBearer(options =>
{
    options.Authority = "https://login.microsoftonline.com/<MY_TENANT>/";
    options.Audience = "api://<MY_API_CLIENT_ID>";
});

builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("Reader", policy => policy.RequireRole("WeatherForecast.Reader"));
});

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}
app.UseAuthentication();
app.UseAuthorization();
app.UseHttpsRedirection();
app.UseCors(cp => cp
    .AllowAnyHeader()
    .SetIsOriginAllowed(origin => true)
    .AllowCredentials()
    );

var summaries = new[]
{
    "Freezing", "Bracing", "Chilly", "Cool", "Mild", "Warm", "Balmy", "Hot", "Sweltering", "Scorching"
};

app.MapGet("/weatherforecast", () =>
{
    var forecast =  Enumerable.Range(1, 5).Select(index =>
        new WeatherForecast
        (
            DateOnly.FromDateTime(DateTime.Now.AddDays(index)),
            Random.Shared.Next(-20, 55),
            summaries[Random.Shared.Next(summaries.Length)]
        ))
        .ToArray();
    return forecast;
})
.WithName("GetWeatherForecast")
.RequireAuthorization("Reader")
.WithOpenApi();

app.Run();

record WeatherForecast(DateOnly Date, int TemperatureC, string? Summary)
{
    public int TemperatureF => 32 + (int)(TemperatureC / 0.5556);
}
```

The implementation is straightforward and requires minimal additional code. We add a **Reader** policy in the `AddAuthorization` method and apply that policy in the `RequireAuthorization` method. The SPA does not need any changes to function as expected. Press F5 to start the API, and run `npx serve .` inside the `Client` folder to launch the SPA:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1741136091485/9437bef8-67fc-416d-9c37-c4bbcd0ea60c.png align="center")

> The library [Microsoft.Identity.Web](https://github.com/AzureAD/microsoft-identity-web) can be used instead of directly setting up the `AddJwtBearer` method. The main difference is that this library offers a broader integration between our applications and Azure Entra ID. Examples can be found [here](https://github.com/Azure-Samples/active-directory-aspnetcore-webapp-openidconnect-v2/blob/master/5-WebApp-AuthZ/5-1-Roles/README.md).

All the code can be found [here](https://github.com/raulnq/azure-entraid-scopes/tree/auth). Thanks, and happy coding.