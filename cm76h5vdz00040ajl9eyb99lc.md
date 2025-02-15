---
title: "Securing .NET API with Microsoft Entra ID"
datePublished: Sat Feb 15 2025 17:31:50 GMT+0000 (Coordinated Universal Time)
cuid: cm76h5vdz00040ajl9eyb99lc
slug: securing-net-api-with-microsoft-entra-id
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1739568214478/438ebcf2-23b3-40c4-bc7a-5b2afa962f8f.png
tags: dotnet, oauth2, microsoft-entra-id

---

In this article, we delve into securing .NET APIs using Microsoft Entra ID, with a focus on scenarios where the API's client is a Single-Page Application. Following the OAuth 2.0 guidelines, we will implement the [**Authorization Code Flow with Proof Key for Code Exchange (PKCE)**](https://auth0.com/docs/get-started/authentication-and-authorization-flow/authorization-code-flow-with-pkce) as our chosen authorization mechanism.

## API App Registration

[L](https://auth0.com/docs/get-started/authentication-and-authorization-flow/authorization-code-flow-with-pkce)et's start creating the App registration for our API.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1739571838606/91fbd644-060f-4155-8f58-c197ad0dcf96.png align="center")

Go to the **Expose an API** option and click **Add a scope:**

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1739572625130/5ed7844a-83f1-4994-b8ed-c4d4aa61613c.png align="center")

At this point, we can use any name for our scopes. In this case, we created a **read** scope. Before adding our first scope, Microsoft Entra ID might ask us to confirm the Application ID URI. Just use the default value, the `ClientID` of our App Registration.

> Scope is a mechanism in OAuth 2.0 to limit an application's access to a user's account. An application can request one or more scopes, this information is then presented to the user in the consent screen, and the access token issued to the application will be limited to the scopes granted.

## SPA App Registration

Let's create the app registration for our SPA:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1739573270762/fc9fcdf5-6da0-4103-89c4-b5bd12afa7be.png align="center")

Go to the **Authentication** option, click **Add a platform**, and select **Single-page application**:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1739572349192/297910b3-b50b-4e01-9f67-28e78baf34b1.png align="center")

Locally, the SPA will run on port 3000, which is why we are using that as the redirect URI. Go to the **API permissions** option, click **Add a permission**, select **APIs my organization uses**, and search for **My API**:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1739573533509/addc1059-fe26-4c9a-9a45-237476295f11.png align="center")

Select the **read** permission. Once the permission is added, click on the option **Grant admin consent**:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1739573703159/e05ccbc3-514e-4a04-a365-afd7caccc3cb.png align="center")

## API

Run the following commands:

```powershell
dotnet new webapi -n MyWebApi
dotnet new sln -n Sandbox
dotnet sln add --in-root MyWebApi
```

Open the `Program.cs` file and update the content as follows:

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
builder.Services.AddAuthorization();
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
.RequireAuthorization()
.WithOpenApi();

app.Run();

record WeatherForecast(DateOnly Date, int TemperatureC, string? Summary)
{
    public int TemperatureF => 32 + (int)(TemperatureC / 0.5556);
}
```

In the code above, the `AddAuthentication` method sets the default authentication scheme to JWT. The `AddJwtBearer` method specifies the authority and audience to validate the token. Remember to use the `AddAuthorization` method, along with the two middlewares: `UseAuthentication` and `UseAuthorization`. Another important piece of the code is the `RequireAuthorization` method under the `/weatherforecast` endpoint. Press F5 to start the API.

## SPA

At the solution level, create a `Client` folder and include an `index.html` file with the following content:

```xml
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Azure Entra ID Authentication</title>
</head>
<body>
    <h1>SPA with Azure Entra ID</h1>
    <button id="login">Login</button>
    <button id="callAPI" style="display:none;">Call API</button>
    <p id="userInfo"></p>
    <p id="token"></p>
    <p id="response"></p>
    <script src="https://alcdn.msauth.net/browser/2.32.1/js/msal-browser.min.js"></script>
    <script>
        const msalConfig = {
            auth: {
                clientId: "<MY_SPA_CLIENT_ID>",
                authority: "https://login.microsoftonline.com/<MY_TENANT>",
                redirectUri: "http://localhost:3000"
            }
        };

        const client = new msal.PublicClientApplication(msalConfig);

        let account;

        async function login() {
            try {
                const loginRequest = {
                    scopes: ["openid", "profile", "offline_access"]
                };
                const loginResponse = await client.loginPopup(loginRequest);
                console.log("Login successful:", loginResponse);
                document.getElementById("userInfo").innerText = `Hello, ${loginResponse.account.username}`;
                document.getElementById("callAPI").style.display = "block";
                account = loginResponse.account;
            } catch (error) {
                console.error("Login failed:", error);
            }
        }

        async function getAccessToken() {
            try {
                const tokenRequest = {
                    scopes: ["api://<MY_API_CLIENT_ID>/read"],
                    account: account
                };
                const tokenResponse = await client.acquireTokenSilent(tokenRequest);
                console.log(tokenResponse);
                document.getElementById("token").innerText = tokenResponse.accessToken;
                return tokenResponse.accessToken;
            } catch (error) {
                console.error("Token acquisition failed:", error);
            }
        }

        async function callAPI() {
            const token = await getAccessToken();
            if (!token) return;

            const response = await fetch("https://localhost:7063/weatherforecast", {
                method: "GET",
                headers: {
                    "Authorization": `Bearer ${token}`
                }
            });
            console.log("API Response:", response)

            const data = await response.text();

            document.getElementById("response").innerText = data; 
        }

        document.getElementById("login").addEventListener("click", login);
        document.getElementById("callAPI").addEventListener("click", callAPI);
    </script>
</body>
</html>
```

Our SPA is built with plain JavaScript. There are two buttons: one for logging in and another for calling the API. We use the [MSAL 2.0](https://github.com/AzureAD/microsoft-authentication-library-for-js/tree/dev/lib/msal-browser) library, which defaults to using the authorization code flow with PKCE. To run this application, execute `npx serve .` inside the `Client` folder (make sure [Node.js](https://nodejs.org/en) is installed).

> `serve` helps you serve a static site, single page application or just a static file

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1739577151186/d2339c4d-26bc-4128-a44b-eb06260e9913.png align="center")

Go to `http://localhost:3000` and start using the application:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1739577336613/0b29562f-2e69-401a-80c9-74ffcb2e85ca.png align="center")

## Lessons Learned Requesting Tokens

* All the scopes must belong to a single resource (app registration).
    
* The resource owner of the scopes must match the audience specified in our API.
    
* When no scope is specified, our token's audience defaults to `00000003-0000-0000-c000-000000000000`, which corresponds to Microsoft Graph.
    
* An app registration can have scopes from multiple resources.
    

All the code can be found [here](https://github.com/raulnq/azure-entraid-scopes). Thanks, and happy coding.