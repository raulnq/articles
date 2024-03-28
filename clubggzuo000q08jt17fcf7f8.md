---
title: "Securing .NET APIs with Amazon Cognito"
datePublished: Thu Mar 28 2024 16:35:10 GMT+0000 (Coordinated Universal Time)
cuid: clubggzuo000q08jt17fcf7f8
slug: securing-net-apis-with-amazon-cognito
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1711497125369/615fa9ea-814e-413d-843b-b76e10f78bef.png
tags: authentication, aws, net, amazon-cognito

---

In this article, we delve into securing .NET APIs with Amazon Cognito, specifically focusing on ensuring authenticated users have the necessary permissions to access resources. Building on the foundation laid in the previous article, [Amazon Cognito Authentication with Hosted UI for](https://blog.raulnq.com/amazon-cognito-authentication-with-hosted-ui-for-aspnet-core-apps) [ASP.NET](http://ASP.NET) [Core Apps](https://blog.raulnq.com/amazon-cognito-authentication-with-hosted-ui-for-aspnet-core-apps), we encourage you to first download the code provided [there](https://github.com/raulnq/aws-cognito/tree/hosted-ui).

## Resource Server

> [The resourc](https://blog.raulnq.com/amazon-cognito-authentication-with-hosted-ui-for-aspnet-core-apps)e server is the OAuth 2.0 term for your API server. The resource server handles authenticated requests after the application has obtained an access token.
> 
> The resource server will be getting requests from applications with an HTTP `Authorization` header containing an access token. The resource server needs to be able to verify the access token to determine whether to process the request, and find the associated user account, etc.

Amazon Cognito allows us to define [Resource Servers](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-define-resource-servers.html?icmpid=docs_cognito_console_help_panel) and associate scopes with them. These scopes can be used to authorize specific actions in our APIs. Our user pool client requests the scopes as part of the authentication process and includes them inside the access token used against the resource server, which can then make decisions based on those scopes. Let's start updating the `template.yaml` file as follows:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31
Description: >
  SAM Template

Resources:
  UserPool:
    Type: AWS::Cognito::UserPool
    Properties:
      UsernameAttributes:
       - email
      UsernameConfiguration: 
        CaseSensitive: false
      Policies:
        PasswordPolicy:
         MinimumLength: 8
         RequireLowercase: false
         RequireNumbers: false
         RequireSymbols: false
         RequireUppercase: false
         TemporaryPasswordValidityDays: 7
      MfaConfiguration: 'OFF'
      AccountRecoverySetting:
        RecoveryMechanisms:
          - Name: verified_email
            Priority: 1
      AdminCreateUserConfig:
        AllowAdminCreateUserOnly: false
      AutoVerifiedAttributes:
        - email
      UserPoolName: "myuserpool"
      UserAttributeUpdateSettings:
        AttributesRequireVerificationBeforeUpdate:
          - email 
      Schema:
        - Name: email
          AttributeDataType: String
          Mutable: false
          Required: true
      EmailConfiguration:
        EmailSendingAccount: COGNITO_DEFAULT

  UserPoolClient:
    Type: "AWS::Cognito::UserPoolClient"
    DependsOn: UserPoolResourceServer
    Properties:
      ClientName: "myclient"
      GenerateSecret: true
      UserPoolId: !Ref UserPool
      ExplicitAuthFlows:
        - ALLOW_USER_SRP_AUTH
        - ALLOW_REFRESH_TOKEN_AUTH 
      SupportedIdentityProviders:
        - COGNITO
      AllowedOAuthFlowsUserPoolClient: true
      AllowedOAuthFlows:
        - code
      AllowedOAuthScopes:
        - email
        - openid
        - profile
        - aws.cognito.signin.user.admin
        - weatherapi/read
      LogoutURLs:
        - "https://localhost:7119/Account/Loggedout" 
      CallbackURLs:
        - "https://localhost:7119/signin-oidc"

  UserDomainPool:
    Type: AWS::Cognito::UserPoolDomain
    Properties:
      Domain: "myuserdomainpoolx95"
      UserPoolId: !Ref UserPool

  UserPoolResourceServer:
    Type: AWS::Cognito::UserPoolResourceServer
    Properties:
      Identifier: weatherapi
      Name: Weather API
      Scopes: 
        - ScopeName: read
          ScopeDescription: "Read access"
      UserPoolId: !Ref UserPool
 
Outputs:
  CognitoUserPoolID:
    Value: !Ref UserPool
    Description: The UserPool ID
  CognitoClientID:
    Value: !Ref UserPoolClient
    Description: The app client
  HostedUIDomain:
    Value: !Ref UserDomainPool
    Description: Hosted UI domain
```

From the original file, we added the [`AWS::Cognito::UserPoolResourceServer`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-cognito-userpoolresourceserver.html) resource:

* `Name`: A unique identifier for the resource server within the user pool.
    
* `Identifier`: A unique identifier for the resource server within the user pool.
    
* `Scopes`: An array that defines the permissions associated with the resource server. Each scope has a name and a description.
    

The final step is to add the new scope to the `AllowedOAuthScopes` property of the `AWS::Cognito::UserPoolClient` resource. The format for the scope is always `ResourceServerIdentifier/ScopeName`. Run the following commands to update the resources:

```yaml
sam build
sam deploy --guided
```

## The target API

Let's set up the API that our application will invoke. Run the following commands:

```powershell
dotnet new webapi -n MyWebApi
dotnet sln add --in-root MyWebApi
dotnet add MyWebApi package Microsoft.AspNetCore.Authentication.JwtBearer
```

Open the `MyWebApi` project and update the `Program.cs` file as follows:

```csharp
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.AspNetCore.Authorization;
using Microsoft.IdentityModel.JsonWebTokens;
using Microsoft.IdentityModel.Tokens;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();
builder.Services.AddAuthentication(options =>
{
    options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
})
.AddJwtBearer(options =>
{
    var configuration = builder.Configuration;
    options.MetadataAddress = $"https://cognito-idp.{configuration["AWS:Region"]}.amazonaws.com/{configuration["AWS:UserPoolId"]}/.well-known/openid-configuration";
    options.TokenValidationParameters = new TokenValidationParameters
    {
        ValidateIssuerSigningKey = true,
        ValidateIssuer = true,
        ValidateLifetime = true,
        ValidateAudience = true,
        ValidAudience = configuration["AWS:UserPoolClientId"],
        AudienceValidator = (audiences, securityToken, validationParameters) =>
        {
            var token = securityToken as JsonWebToken;
            var clientId = token?.GetClaim("client_id").Value;
            return validationParameters.ValidateAudience ? validationParameters.ValidAudience.Equals(clientId) : true;
        }
    };
}
);
builder.Services.AddAuthorization();
var app = builder.Build();
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}
app.UseHttpsRedirection();
app.UseAuthentication();
app.UseAuthorization();
var summaries = new[]
{
    "Freezing", "Bracing", "Chilly", "Cool", "Mild", "Warm", "Balmy", "Hot", "Sweltering", "Scorching"
};
app.MapGet("/weatherforecast", [Authorize] () =>
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
.WithOpenApi();
app.Run();

record WeatherForecast(DateOnly Date, int TemperatureC, string? Summary)
{
    public int TemperatureF => 32 + (int)(TemperatureC / 0.5556);
}
```

The `AddJwtBearer` method sets up the [JSON Web Token](https://datatracker.ietf.org/doc/html/rfc7519) (JWT) validation middleware, which enables handle access tokens:

* `MetadataAddress`: Specifies the well-known [OpenID Connect Discovery](https://openid.net/specs/openid-connect-discovery-1_0.html) endpoint provided by the Amazon Cognito user pool. The authentication middleware will retrieve the metadata from this endpoint to configure itself, including validating the token issuer, signing keys, and other parameters.
    
* `TokenValidationParameters`**:** Provides a range of properties used to configure token validation when working with JWTs:
    
    * `ValidateIssuerSigningKey`: Indicates whether the token's issuer signing key should be validated. The security key is typically used to verify that the JWT token has not been tampered with and was issued by a trusted authority. When set to `true`, the token validation process checks whether the issuer signing key provided by the token matches the key(s) specified in the `IssuerSigningKey` or `IssuerSigningKeys` properties.
        
    * `ValidateIssuer`: Indicates whether the token's issuer (`iss` claim) should be validated. The issuer represents the entity that issued the token, typically an identity provider or authentication service. When set to `true`, the token validation process checks whether the issuer of the JWT token matches the value specified in the `ValidIssuer` property.
        
    * `ValidateLifetime`: Indicates whether the token's lifetime should be validated. The lifetime of a JWT token is determined by the `exp` (expiration time) and `nbf` (not before) claims included in the token. When set to `true`, the token validation process checks whether the current time falls within the validity period defined by the expiration time and not before claims.
        
    * `ValidateAudience`: Indicates whether the token's audience should be validated. The audience represents the intended recipient or audience for which the JWT token was issued. When set to `true`, the token validation process checks whether the audience (`aud` claim) of the JWT token matches the expected audience specified in the `ValidAudience` property. According to the OAuth 2.0 specification, the `aud` claim doesn't have to be present in JWT tokens. That's why Amazon Cognito does not include the claim; we can skip the validation entirely or perform a custom one by using the audience from the `client_id` claim.
        
    * `ValidAudience`: The audience that tokens must have to be considered valid. The audience represents the intended recipient of the token, typically a client application or resource server authorized to accept and process the token. The property `ValidAudiences` accepts an array of audience values.
        
    * `AudienceValidator`: A delegate for validating the audience of a token. The delegate returns a boolean value indicating whether the audience claims in the token are considered valid according to custom validation logic.
        

Don't forget to add the `AddAuthorization` method and the corresponding `UseAuthentication` and `UseAuthorization` middleware. The last step is adding the `[Authorize]` attribute to secure our endpoint. Open the `appsettings.json` file to update its content as follows:

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*",
  "AWS": {
    "Region": "<MY_REGION>",
    "UserPoolClientId": "<MY_CLIENT_ID>",
    "UserPoolId": "<MY_USER_POOL_ID>"
  }
}
```

## The application client

Let's update the application client to invoke the target API. We will use the [Duende.AccessTokenManagement](https://github.com/DuendeSoftware/Duende.AccessTokenManagement/wiki) library to automatically manage the access token's lifecycle using a refresh token. Run the following command:

```json
dotnet add MyWebApp package Duende.AccessTokenManagement.OpenIdConnect
dotnet add MyWebApp Microsoft.IdentityModel.Protocols.OpenIdConnect
```

Create the `Models/WeatherViewModel.cs` file with the following content:

```csharp
public class WeatherViewModel
{
    public int TemperatureC { get; set; }
    public int TemperatureF { get; set; }
    public string? Summary { get; set; }
    public DateOnly @model WeatherViewModel[]
@{
    ViewData["Title"] = "Weather";
}
<h1>@ViewData["Title"]</h1>

<table class="table table-bordered">
    <tr>
        <th>Date</th>
        <th>Summary</th>
        <th>TemperatureC</th>
        <th>TemperatureF</th>
    </tr>
    @foreach (var item in Model)
    {
        <tr>
            <td>
                @item.Date
            </td>
            <td>
                @item.Summary
            </td>
            <td>
                @item.TemperatureC
            </td>
            <td>
                @item.TemperatureF
            </td>
        </tr>
    }
</table>{ get; set; }
}
```

Open the `Controllers/HomeController.cs` and update the content as follows:

```csharp
public class HomeController : Controller
{
    private readonly ILogger<HomeController> _logger;

    private readonly IHttpClientFactory _httpClientFactory;

    public HomeController(ILogger<HomeController> logger, 
        IHttpClientFactory httpClientFactory)
    {
        _logger = logger;
        _httpClientFactory = httpClientFactory;
    }

    public IActionResult Index()
    {
        return View();
    }

    public IActionResult Privacy()
    {
        return View();
    }

    public async Task<IActionResult> GetWeather()
    {
        var client = _httpClientFactory.CreateClient("api");
        var response = await client.GetAsync("weatherforecast");
        response.EnsureSuccessStatusCode();
        var models = await response.Content.ReadFromJsonAsync<WeatherViewModel[]>();
        return View(models);
    }

    [ResponseCache(Duration = 0, Location = ResponseCacheLocation.None, NoStore = true)]
    public IActionResult Error()
    {
        return View(new ErrorViewModel { RequestId = Activity.Current?.Id ?? HttpContext.TraceIdentifier });
    }
}
```

We inject the `IHttpClientFactory` dependency into the controller and create a new `GetWeather` method to call the target API. As we can see, there is no special code related to the tokens because it's handled behind the scenes by the library. Create the `View/GetWeather.cshtml` file with the following content:

```csharp
@model WeatherViewModel[]
@{
    ViewData["Title"] = "Weather";
}
<h1>@ViewData["Title"]</h1>

<table class="table table-bordered">
    <tr>
        <th>Date</th>
        <th>Summary</th>
        <th>TemperatureC</th>
        <th>TemperatureF</th>
    </tr>
    @foreach (var item in Model)
    {
        <tr>
            <td>
                @item.Date
            </td>
            <td>
                @item.Summary
            </td>
            <td>
                @item.TemperatureC
            </td>
            <td>
                @item.TemperatureF
            </td>
        </tr>
    }
</table>
```

Open the `Shared/_Layout.cshtml` file and update the content as follows:

```xml
@using Microsoft.AspNetCore.Identity
@using Amazon.Extensions.CognitoAuthentication
@using System.Security.Claims

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>@ViewData["Title"] - MyWebApp</title>
    <link rel="stylesheet" href="~/lib/bootstrap/dist/css/bootstrap.min.css" />
    <link rel="stylesheet" href="~/css/site.css" asp-append-version="true" />
    <link rel="stylesheet" href="~/MyWebApp.styles.css" asp-append-version="true" />
</head>
<body>
    <header>
        <nav class="navbar navbar-expand-sm navbar-toggleable-sm navbar-light bg-white border-bottom box-shadow mb-3">
            <div class="container-fluid">
                <a class="navbar-brand" asp-area="" asp-controller="Home" asp-action="Index">MyWebApp</a>
                <button class="navbar-toggler" type="button" data-bs-toggle="collapse" data-bs-target=".navbar-collapse" aria-controls="navbarSupportedContent"
                        aria-expanded="false" aria-label="Toggle navigation">
                    <span class="navbar-toggler-icon"></span>
                </button>
                <div class="navbar-collapse collapse d-sm-inline-flex justify-content-between">
                    <ul class="navbar-nav flex-grow-1">
                        <li class="nav-item">
                            <a class="nav-link text-dark" asp-area="" asp-controller="Home" asp-action="Index">Home</a>
                        </li>
                        <li class="nav-item">
                            <a class="nav-link text-dark" asp-area="" asp-controller="Home" asp-action="Privacy">Privacy</a>
                        </li>
                    </ul>
                    <ul class="navbar-nav">
                        @if (User.Identity.IsAuthenticated)
                        {
                            <li class="nav-item">
                                <a class="nav-link text-dark" asp-area="" asp-controller="Home" asp-action="GetWeather">Weather</a>
                            </li>
                            <li class="nav-item">
                                <span class="nav-link text-dark">Hello @User.FindFirstValue(ClaimTypes.Email)!</span>
                            </li>
                            <li class="nav-item">
                                <a class="nav-link text-dark" asp-area="" asp-controller="Account" asp-action="Logout">Logout</a>
                            </li>
                        }
                        else
                        {
                            <li class="nav-item">
                                <a class="nav-link text-dark" id="login" asp-area="" asp-controller="Account" asp-action="Login" asp-route-returnUrl="@Url.Action("Index", "Home", new { area = "" })">Login</a>
                            </li>
                        }
                    </ul>
                </div>
            </div>
        </nav>
    </header>
    <div class="container">
        <main role="main" class="pb-3">
            @RenderBody()
        </main>
    </div>

    <footer class="border-top footer text-muted">
        <div class="container">
            &copy; 2024 - MyWebApp - <a asp-area="" asp-controller="Home" asp-action="Privacy">Privacy</a>
        </div>
    </footer>
    <script src="~/lib/jquery/dist/jquery.min.js"></script>
    <script src="~/lib/bootstrap/dist/js/bootstrap.bundle.min.js"></script>
    <script src="~/js/site.js" asp-append-version="true"></script>
    @await RenderSectionAsync("Scripts", required: false)
</body>
</html>
```

Here, we're adding a link to call the API once the user is authenticated. Open the `Program.cs` file and update the content as follows:

```csharp
using Microsoft.AspNetCore.Authentication.Cookies;
using Microsoft.AspNetCore.Authentication.OpenIdConnect;
using Microsoft.IdentityModel.Protocols.OpenIdConnect;

var builder = WebApplication.CreateBuilder(args);

var configuration = builder.Configuration;

builder.Services.AddControllersWithViews();

builder.Services.AddAuthentication(options =>
{
    options.DefaultAuthenticateScheme = CookieAuthenticationDefaults.AuthenticationScheme;
    options.DefaultSignInScheme = CookieAuthenticationDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = OpenIdConnectDefaults.AuthenticationScheme;
})
.AddCookie()
.AddOpenIdConnect(options =>
{
    options.ResponseType = OpenIdConnectResponseType.Code;
    options.MetadataAddress = $"https://cognito-idp.{configuration["AWS:Region"]}.amazonaws.com/{configuration["AWS:UserPoolId"]}/.well-known/openid-configuration";
    options.ClientId = configuration["AWS:UserPoolClientId"];
    options.ClientSecret = configuration["AWS:UserPoolClientSecret"];
    options.UsePkce = true;
    options.Events = new OpenIdConnectEvents()
    {
        OnRedirectToIdentityProviderForSignOut = OnRedirectToIdentityProviderForSignOut
    };
    options.Scope.Clear();
    options.Scope.Add("openid");
    options.Scope.Add("email");
    options.Scope.Add("aws.cognito.signin.user.admin");
    options.Scope.Add("profile");
    options.Scope.Add("weatherapi/read");
    options.SaveTokens = true;

    Task OnRedirectToIdentityProviderForSignOut(RedirectContext context)
    {
        context.ProtocolMessage.Scope = "openid";
        context.ProtocolMessage.ResponseType = OpenIdConnectResponseType.Code;
        var cognitoDomain = $"https://{configuration["AWS:Domain"]}.auth.{configuration["AWS:Region"]}.amazoncognito.com" ;
        var clientId = configuration["AWS:UserPoolClientId"]; ;
        var logoutUrl = $"{context.Request.Scheme}://{context.Request.Host}{configuration["AWS:AppSignOutUrl"]}";
        context.ProtocolMessage.IssuerAddress = $"{cognitoDomain}/logout?client_id={clientId}&logout_uri={logoutUrl}&redirect_uri={logoutUrl}";
        context.Properties.Items.Remove(CookieAuthenticationDefaults.AuthenticationScheme);
        context.Properties.Items.Remove(OpenIdConnectDefaults.AuthenticationScheme);
        return Task.CompletedTask;
    }
});

builder.Services.AddOpenIdConnectAccessTokenManagement();

builder.Services.AddHttpClient("api", client =>
{
    client.BaseAddress = new Uri("https://localhost:7064/");
})
.AddUserAccessTokenHandler();

var app = builder.Build();

if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/Home/Error");
    app.UseHsts();
}
app.UseHttpsRedirection();
app.UseStaticFiles();
app.UseRouting();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");

app.Run();
```

The changes here are:

* In the `AddOpenIdConnect` method, we added the new scope `weatherapi/read` to the list.
    
* We added the `AddOpenIdConnectAccessTokenManagement` method to utilize the Duende library.
    
* A delegate handler is added to the HTTP client using the `AddUserAccessTokenHandler` method to include the access token every time it is used.
    

And that's it. Run the application and enjoy our secure API.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1711640769316/e3b351ca-f083-4683-893e-606470e39a39.png align="center")

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1711640803612/c8f5a4a6-2e7a-471e-b400-48e2a8020563.png align="center")

All the code can be found [here](https://github.com/raulnq/aws-cognito/tree/resource-server). Thanks, and happy coding.