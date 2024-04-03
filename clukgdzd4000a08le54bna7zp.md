---
title: "How to Implement Authorization in .NET APIs with Amazon Cognito"
datePublished: Wed Apr 03 2024 23:42:45 GMT+0000 (Coordinated Universal Time)
cuid: clukgdzd4000a08le54bna7zp
slug: how-to-implement-authorization-in-net-apis-with-amazon-cognito
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1712157410040/ccfecaa5-d87f-478f-af3f-8224366c87e6.png
tags: aws, authorization, net, amazon-cognito

---

Authorization is the process that determines what users can do after they are authenticated. In the .NET ecosystem, it's straightforward to implement the most common approaches, such as role-based access control (RBAC) or attribute-based access control (ABAC). To show how this works, we will modify the code from the article [Securing .NET APIs with Amazon Cognito](https://blog.raulnq.com/securing-net-apis-with-amazon-cognito) to use these methods. The starting code is available for download [here](https://github.com/raulnq/aws-cognito/tree/resource-server).

## Role-based access control

In RBAC, access permissions are assigned to roles, and users are assigned to them. Open the solution, go to the `MyWebApi` project, and update the endpoint definition in the `Program.cs` file as shown below:

```csharp
app.MapGet("/weatherforecast", [Authorize(Roles = "Premium")] () =>
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
```

We are authorizing the endpoint only for users in the `Premium` role. Open the `template.yaml` file and add the following resource:

```yaml
  UserPoolGroup:
    Type: AWS::Cognito::UserPoolGroup
    Properties:
      GroupName: Premium
      Precedence: 0
      UserPoolId: !Ref UserPool
```

The resource [`AWS::Cognito::UserPoolGroup`](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-cognito-userpoolgroup.html) will create a group in Amazon Cognito. A group is a collection of users within a pool that shares permissions. Run the following commands to update the resources:

```powershell
sam build
sam deploy --guided
```

[Add a user to the group](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-user-groups.html) by running the following command:

```powershell
aws cognito-idp admin-add-user-to-group --user-pool-id <MY_USER_POOL_ID> --username <MY_USER_NAME> --group-name Premium
```

As a result, the access and ID tokens both include a `cognito:groups` claim that contains the user's groups. Go to the `MyWebApi` project and update the `AddJwtBearer` method in the `Program.cs` file as shown below:

```csharp
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
        RoleClaimType = "cognito:groups",
        AudienceValidator = (audiences, securityToken, validationParameters) =>
        {
            var token = securityToken as JsonWebToken;
            var clientId = token?.GetClaim("client_id").Value;
            return validationParameters.ValidateAudience ? validationParameters.ValidAudience.Equals(clientId) : true;
        }
    };
});
```

The default role's claim type is `http://schemas.microsoft.com/ws/2008/06/identity/claims/role`. Therefore, we need to change the `RoleClaimType` property to `cognito:groups`. Run the application to see the RBAC authorization in action.

## Attribute-based access control

In ABAC, access control policies are defined based on the user's attributes. Go to the `MyWebApi` project and update the `AddAuthorization` method in the `Program.cs` file as follows:

```csharp
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("Premium", policy => policy.RequireClaim("ispremium", "true"));
});
```

We are defining a policy named `Premium` that requires a claim named `ispremium` with the value set to `true`. Then, update the endpoint definition as shown below:

```csharp
app.MapGet("/weatherforecast", [Authorize(Policy = "Premium")] () =>
{
    var forecast = Enumerable.Range(1, 5).Select(index =>
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
```

We are authorizing the endpoint only for users who meet the `Premium` policy. Next, we need to add a new attribute to the schema in the user pool. Once the attribute is assigned to the user, it will only be available in the ID token. A [pre-token generation lambda trigger](https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-lambda-pre-token-generation.html#user-pool-lambda-pre-token-generation-accesstoken) is needed to include the claim in the access token. Run the following commands to create the lambda function:

```powershell
dotnet new lambda.EmptyFunction -n MyLambda -o .
dotnet add src/MyLambda package Amazon.Lambda.CognitoEvents
dotnet sln add --in-root src/MyLambda
```

Open the `Function.cs` file and replace its content with:

```csharp
using Amazon.Lambda.CognitoEvents;
using Amazon.Lambda.Core;

[assembly: LambdaSerializer(typeof(Amazon.Lambda.Serialization.SystemTextJson.DefaultLambdaJsonSerializer))]
namespace MyLambda;

public class Function
{
    public CognitoPreTokenGenerationV2Event FunctionHandler(CognitoPreTokenGenerationV2Event input, ILambdaContext context)
    {
        var isPremium = false;
        if (input.Request.UserAttributes.ContainsKey("custom:ispremium"))
        {
            isPremium = input.Request.UserAttributes["custom:ispremium"].Equals("true", StringComparison.InvariantCultureIgnoreCase);
        }

        input.Response = new CognitoPreTokenGenerationV2Response()
        {
            ClaimsAndScopeOverrideDetails = new ClaimsAndScopeOverrideDetails()
            {
                AccessTokenGeneration = new AccessTokenGeneration()
                {
                    ClaimsToAddOrOverride = new Dictionary<string, string>() { { "ispremium", isPremium.ToString().ToLower() } }
                },
                GroupOverrideDetails = input.Request.GroupConfiguration
            },
        };
        return input;
    }
}
```

The `CognitoPreTokenGenerationV2Event` includes a `Request` property that contains all the user's attributes and a `Response` property with an array of claims to be added to the access token. One thing to notice is that we are copying the original group information to the request to keep the `cognito:groups` in the access token. Open the `template.yaml` file and add the following resource:

```yaml
  PreTokenGenerationFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: MyLambda::MyLambda.Function::FunctionHandler
      CodeUri: ./src/MyLambda/
      Runtime: dotnet8
      Timeout: 30
      MemorySize: 512
      Architectures:
        - x86_64

  CognitoPermission:
    Type: AWS::Lambda::Permission
    Properties:
      Action: lambda:InvokeFunction
      FunctionName: !Ref PreTokenGenerationFunction
      Principal: cognito-idp.amazonaws.com
      SourceArn: !GetAtt UserPool.Arn
```

The `AWS::Serverless::Function` resource deploys the lambda function, and the `AWS::Lambda::Permission` lets Amazon Cognito invoke it. Update the following resource in the same file as shown:

```yaml
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
      - Name: ispremium
        AttributeDataType: Boolean
        Mutable: true
        Required: false
    EmailConfiguration:
      EmailSendingAccount: COGNITO_DEFAULT
    UserPoolAddOns:
      AdvancedSecurityMode: AUDIT
    LambdaConfig:
      PreTokenGenerationConfig:
        LambdaArn: !GetAtt PreTokenGenerationFunction.Arn
        LambdaVersion: V2_0  
```

Here, we are adding the attribute `ispremium` to the user's schema. We need the pre-token generation lambda version 2 to modify the access token. The feature requires the [advanced security features](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pool-settings-advanced-security.html) enabled (`UserPoolAddOns` property). Finally, in the `PreTokenGenerationConfig` property, we referenced our lambda function and the version.

> * The advanced security features generate extra [costs](https://aws.amazon.com/cognito/pricing/?nc1=h_ls).
>     
> * The [`Cognito`](https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-lambda-pre-token-generation.html) EventSource of the `AWS::Serverless::Function` resource does not support `PreTokenGeneration` version 2 as a trigger.
>     

Run the following command to update the resources:

```powershell
sam build
sam deploy --guided
```

Add the attribute to the user with the following command:

```powershell
aws cognito-idp admin-update-user-attributes --user-pool-id <MY_USER_POOL_ID> --username <MY_USER_NAME> --user-attributes Name="custom:ispremium",Value="true"
```

Run the application to see the ABAC authorization in action. All the code can be found [here](https://github.com/raulnq/aws-cognito/tree/authorization). Thanks, and happy coding.