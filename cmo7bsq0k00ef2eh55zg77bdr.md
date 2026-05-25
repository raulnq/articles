---
title: "AWS API Gateway HTTP API: Check List"
datePublished: 2026-04-20T15:04:25.273Z
cuid: cmo7bsq0k00ef2eh55zg77bdr
slug: aws-api-gateway-http-api-check-list
cover: https://cdn.hashnode.com/uploads/covers/625b81432f84e0d4fc33dca5/95eeaa3b-d9cf-4c5a-80d5-87d069a7d58b.png
tags: lambda, aws, dotnet, api-gateway

---

HTTP API is the right default choice for the majority of .NET serverless workloads. At 70% less cost than REST API, lower latency, and native JWT authorization, it covers most production requirements without the operational overhead of its older sibling. But "simpler" does not mean "automatically safe or production-ready."

This guide covers seven areas of HTTP API excellence for .NET. It assumes you are familiar with AWS SAM or CloudFormation and focuses on the *why* behind each practice as much as the *how*.

## When to Choose HTTP API

HTTP API is the right choice when you need:

| Feature | HTTP API |
| --- | --- |
| Cost per million requests | **$1.00** |
| Native JWT authorizer (Cognito / OIDC) | ✅ |
| Lambda proxy integration | ✅ |
| Stage-level throttling | ✅ |
| Mutual TLS | ✅ |
| CORS configuration | ✅ |
| VPC Link (v2) | ✅ |
| X-Ray tracing (gateway segment) | ❌ — use REST AP |
| Per-route throttling | ❌ — use REST API |
| Usage plans + API keys | ❌ — use REST API |
| Response caching | ❌ — use CloudFront |
| Request body validation | ❌ — validate in Lambda |
| AWS service integrations | ❌ — use REST API |
| Private API (VPC endpoint) | ❌ — use REST API |

Move to REST API only when you hit a concrete missing feature. Do not use REST API by default.

* * *

## Security

### Use the Native JWT Authorizer for Token-Based Authentication

**What it is:** HTTP API has a built-in JWT authorizer that validates Bearer tokens from Cognito User Pools or any standards-compliant OIDC/OAuth2 provider. Signature verification, expiry, audience, and issuer checks all happen inside API Gateway before your Lambda is ever invoked.

**Why it matters:** Without a native authorizer, every request that reaches your Lambda requires your .NET code to validate the token, consuming billed Lambda duration on every call — including ones that should be rejected. A misconfigured manual validation (e.g., not checking the `aud` claim) is a common source of auth bypasses. The native authorizer prevents this class of vulnerability entirely.

```yaml
Resources:
  MyApi:
    Type: AWS::Serverless::HttpApi
    Properties:
      Auth:
        DefaultAuthorizer: CognitoJwtAuth
        Authorizers:
          CognitoJwtAuth:
            IdentitySource: $request.header.Authorization
            JwtConfiguration:
              issuer: !Sub "https://cognito-idp.${AWS::Region}.amazonaws.com/${UserPoolId}"
              audience:
                - !Ref UserPoolClientId

  MyFunction:
    Type: AWS::Serverless::Function
    Properties:
      Events:
        GetOrders:
          Type: HttpApi
          Properties:
            ApiId: !Ref MyApi
            Path: /orders
            Method: GET
            Auth:
              Authorizer: CognitoJwtAuth
```

The authorizer context (claims from the JWT) is available inside your Lambda via `request.RequestContext.Authorizer.Jwt.Claims`:

```csharp
public APIGatewayHttpApiV2ProxyResponse FunctionHandler(
    APIGatewayHttpApiV2ProxyRequest request,
    ILambdaContext context)
{
    // Extract claims injected by the JWT authorizer
    var userId = request.RequestContext.Authorizer.Jwt.Claims["sub"];
    var tenantId = request.RequestContext.Authorizer.Jwt.Claims["custom:tenantId"];
    // ...
}
```

* * *

### Use a Lambda Authorizer for Custom Auth Logic

**What it is:** HTTP API supports Lambda Authorizers using the v2.0 payload format. The authorizer receives a structured event with the full request context and returns a simple boolean `isAuthorized` plus a context map — not the complex IAM policy document that REST API authorizers must return.

**Why it matters:** The native JWT authorizer handles the common case, but many production systems need logic it cannot provide: token introspection against a remote OAuth2 server, API key validation with database lookups, or multi-tenant role resolution. The v2.0 payload format is considerably simpler to implement in .NET than the REST API authorizer format.

Enable result caching to avoid a Lambda invocation on every request:

```yaml
Resources:
  MyApi:
    Type: AWS::Serverless::HttpApi
    Properties:
      Auth:
        DefaultAuthorizer: CustomAuth
        Authorizers:
          CustomAuth:
            AuthorizerPayloadFormatVersion: "2.0"
            EnableSimpleResponses: true          # Returns {isAuthorized: bool, context: {}}
            IdentitySource: $request.header.Authorization
            FunctionArn: !GetAtt AuthorizerFunction.Arn
```

The .NET authorizer implementation with the simple response format. There are no dedicated SDK types for the HTTP API authorizer — the input matches the v2 proxy request shape, and the simple response is a POCO you define:

```csharp
// No dedicated authorizer types exist in Amazon.Lambda.APIGatewayEvents.
// The format 2.0 authorizer input matches APIGatewayHttpApiV2ProxyRequest.
// Define your own simple response POCO — the property names must match exactly.
public class SimpleAuthorizerResponse
{
    [JsonPropertyName("isAuthorized")]
    public bool IsAuthorized { get; set; }

    [JsonPropertyName("context")]
    public Dictionary<string, object> Context { get; set; } = new();
}

public class AuthorizerFunction
{
    public SimpleAuthorizerResponse FunctionHandler(
        APIGatewayHttpApiV2ProxyRequest request,   // Same type as a regular handler
        ILambdaContext context)
    {
        var token = request.Headers["authorization"];
        var (isValid, claims) = ValidateToken(token);

        return new SimpleAuthorizerResponse
        {
            IsAuthorized = isValid,
            Context = new Dictionary<string, object>
            {
                { "userId", claims?.UserId ?? "" },
                { "tenantId", claims?.TenantId ?? "" }
            }
        };
    }
}
```

* * *

### Protect Your HTTP API with WAF via CloudFront

**What it is:** AWS WAF cannot be attached directly to an HTTP API stage — it only supports REST API stages, CloudFront distributions, ALBs, AppSync, and Cognito User Pools. To add WAF protection to an HTTP API, place a CloudFront distribution in front of it and attach the WAF ACL there.

**Why it matters:** Without WAF, a public HTTP API has no protection against SQLi, XSS, known malicious IPs, or request floods. A flood will invoke your Lambda and bill you even if your JWT authorizer rejects every request. CloudFront + WAF blocks abuse at the edge before any API Gateway or Lambda cost is incurred — and because it runs at the nearest edge location, the protection is actually applied closer to the attacker than a regional WAF on REST API would be.

* * *

### Configure CORS Correctly

**What it is:** HTTP API has a first-class CORS configuration block that adds the appropriate response headers automatically. A wildcard (`*`) in `AllowOrigins` permits any website to call your API from a user's browser.

**Why it matters:** `AllowOrigins: "*"` combined with a JWT authorizer appears safe but is not — it allows any origin to trigger cross-origin requests carrying your users' credentials. Enumerate the exact origins your frontend uses and reject everything else at the gateway, for free, before any Lambda is invoked.

```yaml
Resources:
  MyApi:
    Type: AWS::Serverless::HttpApi
    Properties:
      CorsConfiguration:
        AllowOrigins:
          - "https://app.example.com"
          - "https://staging.example.com"
        AllowMethods:
          - GET
          - POST
          - PUT
          - DELETE
          - OPTIONS
        AllowHeaders:
          - Authorization
          - Content-Type
          - X-Request-Id
        ExposeHeaders:
          - X-Request-Id
        AllowCredentials: false
        MaxAge: 600         # Cache preflight result for 10 minutes
```

> Do not return `Access-Control-Allow-Origin: *` from your Lambda response either — the HTTP API gateway-level CORS configuration takes precedence, but inconsistent headers cause subtle browser issues.

* * *

### Enforce TLS 1.2 on Custom Domains

**What it is:** HTTP API custom domains support two security policies: `TLS_1_0` and `TLS_1_2`. The `TLS_1_2` policy rejects handshakes from clients using TLS 1.0 or 1.1.

**Why it matters:** TLS 1.0 and 1.1 are deprecated, have known vulnerabilities, and are explicitly prohibited by PCI-DSS 3.2+, SOC 2, and HIPAA security frameworks. Setting this is a one-line change with no downside for any modern client or browser.

```yaml
Resources:
  ApiCustomDomain:
    Type: AWS::ApiGatewayV2::DomainName
    Properties:
      DomainName: api.example.com
      DomainNameConfigurations:
        - CertificateArn: !Ref ApiCertificate
          EndpointType: REGIONAL
          SecurityPolicy: TLS_1_2    # Never TLS_1_0
```

* * *

## Throttling

### Set Stage-Level Throttle Limits

**What it is:** HTTP API applies throttling at the stage level using a token bucket: the burst limit is the maximum concurrent requests at any instant, and the rate limit is the sustained average requests per second. HTTP API does not support per-route throttle overrides — all routes share the stage limits.

**Why it matters:** Without throttle limits, a traffic spike or a misbehaving client can scale your Lambda to the account concurrency limit in seconds and generate a bill in minutes. Stage-level throttling caps this at the gateway before Lambda is invoked.

* * *

## Request Validation

### Validate Request Payloads in Your .NET Lambda Handler

**What it is:** HTTP API does not have a native request body validator. Validation of the incoming JSON payload must be performed inside your Lambda function.

**Why it matters:** Without validation, malformed or missing fields reach your business logic and cause unhandled exceptions, resulting in 500 responses and noise in your error alarms. Validating at the handler entry point catches client errors early, returns consistent 400 responses, and keeps your domain logic clean.

* * *

## Integration

### Use Lambda Proxy Integration with the v2.0 Payload Format

**What it is:** HTTP API supports two payload format versions for Lambda proxy integration: v1.0 (identical to REST API format, using `APIGatewayProxyRequest`) and v2.0 (a cleaner, more compact format using `APIGatewayHttpApiV2ProxyRequest`). The v2.0 format is the default for new HTTP API routes.

**Why it matters:** v2.0 is simpler: the body is a plain string (not base64-encoded for text), cookies are parsed automatically, and the request context is richer. Using v1.0 on HTTP API is a common copy-paste mistake from REST API code that works but misses the v2.0 improvements.

```csharp
// v2.0 payload format — use this for HTTP API
public APIGatewayHttpApiV2ProxyResponse FunctionHandler(
    APIGatewayHttpApiV2ProxyRequest request,  // Note: HttpApiV2, not APIGatewayProxyRequest
    ILambdaContext context)
{
    var userId = request.RequestContext.Authorizer.Jwt.Claims["sub"];
    var routeKey = request.RequestContext.RouteKey;   // "GET /orders/{id}"
    var pathParam = request.PathParameters["id"];

    return new APIGatewayHttpApiV2ProxyResponse
    {
        StatusCode = 200,
        Headers = new Dictionary<string, string> { { "Content-Type", "application/json" } },
        Body = JsonSerializer.Serialize(new { userId, pathParam })
    };
}
```

Ensure your SAM function specifies the correct payload format version:

```yaml
Events:
  GetOrder:
    Type: HttpApi
    Properties:
      ApiId: !Ref MyApi
      Path: /orders/{id}
      Method: GET
      PayloadFormatVersion: "2.0"   # Explicit — default for HttpApi but worth stating
```

* * *

### Never Return HTTP 200 for Errors in Your Lambda Handler

**What it is:** HTTP API takes the status code directly from your Lambda's response object. It is entirely possible to catch an exception and return `StatusCode = 200` with an error body. API Gateway will faithfully return a 200 to the client with no indication that anything went wrong.

**Why it matters:** Returning 200 for errors silently breaks every piece of observability infrastructure monitoring HTTP status codes — API Gateway's built-in `4XXError`/`5XXError` metrics, WAF, CloudWatch alarms, and client-side error handling. Always return semantically correct status codes.

```csharp
// Correct error handling pattern
try
{
    var result = await _service.CreateOrderAsync(order, cancellationToken);
    return Response(201, result);
}
catch (ValidationException ex)
{
    return Response(400, new { error = ex.Message });
}
catch (NotFoundException ex)
{
    return Response(404, new { error = ex.Message });
}
catch (Exception ex)
{
    _logger.LogError(ex, "Unhandled error in CreateOrder");
    return Response(500, new { error = "Internal server error" });
}

private static APIGatewayHttpApiV2ProxyResponse Response<T>(int statusCode, T body) =>
    new APIGatewayHttpApiV2ProxyResponse
    {
        StatusCode = statusCode,
        Headers = new Dictionary<string, string> { { "Content-Type", "application/json" } },
        Body = JsonSerializer.Serialize(body)
    };
```

* * *

### Set Integration Timeout Aligned to Your Lambda Timeout

**What it is:** `TimeoutInMillis` on `AWS::ApiGatewayV2::Integration` controls how long API Gateway waits for a response from Lambda before giving up and returning a 504 to the client. The default is 30 seconds (the maximum). You can set it as low as 50ms. It is not exposed on the SAM `HttpApi` event shorthand — you must define the integration resource explicitly in CloudFormation.

**Why it matters:** The integration timeout must always be set **greater than** your Lambda timeout — not the other way around. Here is why:

When Lambda hits its own timeout, it terminates immediately and returns an error response to API Gateway. API Gateway receives that response right away and returns a 504 to the client. No extra waiting happens — Lambda's timeout drives the failure, and the integration timeout is never reached.

The danger is when the integration timeout is **shorter** than the Lambda timeout. In that case, API Gateway gives up and returns 504 to the client while Lambda is still running in the background. Lambda has no idea the client already received an error — it keeps executing, keeps consuming memory, and keeps billing you until it either finishes or hits its own timeout. The result is discarded. You pay for work that produced nothing.

Set the integration timeout to your Lambda timeout plus a 1–2 second margin. This guarantees API Gateway always waits long enough to receive Lambda's own timeout error if one occurs, while bounding the total worst-case wait time close to your Lambda's configured limit.

* * *

### Use VPC Link v2 for Private Backend Integrations

**What it is:** HTTP API uses VPC Link v2 (backed by AWS PrivateLink) to route traffic to private resources inside a VPC — Application Load Balancers, Network Load Balancers, or Cloud Map service discovery endpoints — without traversing the public internet.

**Why it matters:** If your backend is a containerized .NET service on ECS or EKS behind a load balancer, VPC Link is the correct integration pattern. Without it, your load balancer must be internet-facing, relying on security groups alone — a security and compliance risk.

* * *

## Observability

### Enable Access Logging with Structured JSON

**What it is:** HTTP API supports access logging — one JSON record per request — configured via a custom format string. This is the correct logging mode for production: structured, queryable, and inexpensive compared to execution logging (which HTTP API does not support at all).

**Why it matters:** Without access logs you have no per-request visibility into latency, status codes, authorizer errors, or client IP. Using JSON format makes every field queryable with CloudWatch Logs Insights without log parsing.

```yaml
Resources:
  ApiAccessLogGroup:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: !Sub "/aws/apigateway/${MyApi}/access"
      RetentionInDays: 30

  MyApi:
    Type: AWS::Serverless::HttpApi
    Properties:
      AccessLogSettings:
        DestinationArn: !GetAtt ApiAccessLogGroup.Arn
        Format: >-
          {
            "requestId": "$context.requestId",
            "ip": "$context.identity.sourceIp",
            "requestTime": "$context.requestTime",
            "httpMethod": "$context.httpMethod",
            "routeKey": "$context.routeKey",
            "status": "$context.status",
            "responseLength": "$context.responseLength",
            "responseLatency": "$context.responseLatency",
            "integrationLatency": "$context.integrationLatency",
            "userAgent": "$context.identity.userAgent",
            "authorizerError": "$context.authorizer.error",
            "errorMessage": "$context.error.message"
          }
```

Example CloudWatch Logs Insights query to identify slow routes:

```plaintext
fields routeKey, integrationLatency, status
| filter integrationLatency > 1000
| stats avg(integrationLatency) as avgMs, count() as requests by routeKey
| sort avgMs desc
```

* * *

### Set CloudWatch Alarms on 5XX, Latency, and Count

**What it is:** HTTP API publishes CloudWatch metrics per stage: `5XXError`, `Count`, `Latency`, and `IntegrationLatency`. None have alarms by default — you must create them explicitly.

**Why it matters:** Without alarms, a 5XX spike, a latency regression, or a complete request count drop (indicating a broken deployment or DNS issue) go undetected until a user reports a problem. These metrics give you a complete view of API health in seconds rather than hours.

```yaml
Resources:
  Api5xxErrorRateAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: !Sub "${MyApi}-5XXErrorRate"
      ComparisonOperator: GreaterThanThreshold
      Threshold: 0.01   # 1% error rate
      EvaluationPeriods: 2
      DatapointsToAlarm: 2
      TreatMissingData: notBreaching
      Metrics:
        - Id: errors
          MetricStat:
            Metric:
              Namespace: AWS/ApiGateway
              MetricName: 5xx
              Dimensions:
                - Name: ApiId
                  Value: !Ref MyApi
                - Name: Stage
                  Value: $default
            Period: 60
            Stat: Sum
          ReturnData: false
        - Id: count
          MetricStat:
            Metric:
              Namespace: AWS/ApiGateway
              MetricName: Count
              Dimensions:
                - Name: ApiId
                  Value: !Ref MyApi
                - Name: Stage
                  Value: $default
            Period: 60
            Stat: Sum
          ReturnData: false
        - Id: errorRate
          Expression: errors / count
          Label: "5XX Error Rate"
          ReturnData: true
      AlarmActions:
        - !Ref OpsAlertTopic

  ApiP99LatencyAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: !Sub "${MyApi}-P99Latency"
      Namespace: AWS/ApiGateway
      MetricName: Latency
      Dimensions:
        - Name: ApiId
          Value: !Ref MyApi
        - Name: Stage
          Value: $default
      ExtendedStatistic: p99
      Period: 300
      EvaluationPeriods: 3
      Threshold: 5000          # 5 seconds
      ComparisonOperator: GreaterThanThreshold
      TreatMissingData: notBreaching
      AlarmActions:
        - !Ref OpsAlertTopic

  ApiCountDropAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: !Sub "${MyApi}-CountDrop"
      Namespace: AWS/ApiGateway
      MetricName: Count
      Dimensions:
        - Name: ApiId
          Value: !Ref MyApi
        - Name: Stage
          Value: $default
      Statistic: Sum
      Period: 300
      EvaluationPeriods: 2
      Threshold: 1
      ComparisonOperator: LessThanThreshold
      TreatMissingData: breaching   # Absence of traffic is itself an alarm signal
      AlarmActions:
        - !Ref OpsAlertTopic
```

* * *

### Set Log Retention on All Log Groups

**What it is:** The access log group created for HTTP API defaults to "Never Expire," meaning logs accumulate indefinitely. Explicitly set a retention period.

**Why it matters:** CloudWatch Logs storage costs $0.03/GB. A busy HTTP API generating one access log record per request at 500 req/s produces several gigabytes per day. Without a retention policy, this cost grows silently and without bound.

```yaml
Resources:
  ApiAccessLogGroup:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: !Sub "/aws/apigateway/${MyApi}/access"
      RetentionInDays: 30    # Align with your compliance requirements
```

* * *

## Deployment & Stages

### Use Stage Variables to Decouple Configuration from Code

**What it is:** HTTP API stage variables are key-value pairs attached to a stage that decouple environment-specific values from the API definition. `AWS::Serverless::HttpApi` does not expose a `StageVariables` property directly — stage variables must be set on the underlying `AWS::ApiGatewayV2::Stage` resource.

**Why it matters:** Without stage variables, deploying the same API definition to dev, staging, and prod requires hardcoded Lambda ARNs or backend URLs per stack. Stage variables let a single API definition carry different runtime values per environment.

```yaml
Resources:
  MyApi:
    Type: AWS::Serverless::HttpApi
    Properties:
      StageName: prod

  # AWS::Serverless::HttpApi does not expose StageVariables directly.
  # Set them on the underlying AWS::ApiGatewayV2::Stage resource instead.
  MyApiStage:
    Type: AWS::ApiGatewayV2::Stage
    Properties:
      ApiId: !Ref MyApi
      StageName: prod
      AutoDeploy: true
      StageVariables:
        logLevel: ERROR
        backendRegion: !Ref AWS::Region
```

* * *

### Use Custom Domain Names

**What it is:** Every HTTP API gets a default URL in the form `https://{api-id}.execute-api.{region}.amazonaws.com`. A custom domain attaches your own hostname (e.g., `api.example.com`) via an ACM certificate and a Route 53 alias record.

**Why it matters:** The default execute-api URL leaks your AWS region and API ID, changes when you recreate the API, and is unmemorable for consumers. Custom domains decouple your clients from the underlying AWS resource and are required for any production API.

* * *

## Cost Optimization

### Use CloudFront in Front of HTTP API for Caching and Global Performance

**What it is:** HTTP API does not have a built-in response cache. For cacheable GET endpoints (product catalogs, configuration data, reference lookups), a CloudFront distribution in front of the HTTP API provides edge caching, reducing both API Gateway invocations and Lambda costs.

**Why it matters:** Each CloudFront edge cache hit costs a fraction of an API Gateway invocation + Lambda execution. For a GET endpoint receiving 10 million requests/day with a 70% cache hit rate, CloudFront eliminates 7 million Lambda invocations per day. CloudFront also improves global latency by serving cached responses from the edge location nearest to the user.

* * *

### Tag All HTTP API Resources with Cost Allocation Tags

**What it is:** Tag your HTTP API, log groups, and WAF ACLs with consistent key-value pairs (team, environment, service) so AWS Cost Explorer can attribute spending to specific owners.

**Why it matters:** In a shared AWS account all API Gateway charges appear as a single undifferentiated line item. Without tags, there is no way to attribute HTTP API spending to specific teams, features, or environments.

```yaml
Resources:
  MyApi:
    Type: AWS::Serverless::HttpApi
    Properties:
      Tags:
        Team: payments
        Environment: production
        Service: order-api
        CostCenter: "1234"
```

* * *

## Operations

### Return Consistent Error Responses from Your Lambda — HTTP API Has No Custom Gateway Responses

**What it is:** Unlike REST API, HTTP API does not support `AWS::ApiGateway::GatewayResponse` resources. There is no way to customize the body, headers, or format of gateway-level rejections — throttle errors, authorizer failures, and malformed request errors all return a fixed AWS-controlled response body like `{"message":"Forbidden"}`. The only error responses you control are the ones your Lambda returns.

**Why it matters:** Because you cannot override gateway-level error bodies, your strategy shifts: ensure your Lambda returns consistent, well-structured error responses for every case it handles, and configure your gateway-level CORS block to inject the `Access-Control-Allow-Origin` header on all responses including those from authorizer failures and throttle limits (HTTP API's `CorsConfiguration` block does this automatically). Do not try to replicate the gateway response behaviour in Lambda — focus on what you own.

```csharp
// Consistent error response helper — used on every return path
private static APIGatewayHttpApiV2ProxyResponse Response<T>(int statusCode, T body) =>
    new APIGatewayHttpApiV2ProxyResponse
    {
        StatusCode = statusCode,
        Headers = new Dictionary<string, string>
        {
            { "Content-Type", "application/json" }
            // Do NOT add Access-Control-Allow-Origin here.
            // The HTTP API CorsConfiguration block adds it automatically to all responses,
            // including gateway-level rejections that never reach your Lambda.
        },
        Body = JsonSerializer.Serialize(body)
    };
```

> Gateway-level rejections (throttle, auth failure) in HTTP API always return `{"message":"..."}` bodies — you cannot change this. Design your client-side error handling to tolerate this format on 429 and 403 responses, and rely on the HTTP status code rather than the body shape for error classification.

* * *

### Handle Response Compression in Your Lambda or via CloudFront — HTTP API Has No Native Compression

**What it is:** Unlike REST API, HTTP API does not support `MinimumCompressionSize` — there is no gateway-level gzip compression setting on `AWS::ApiGatewayV2::Api`. For HTTP API, compression must be handled either in your Lambda handler or by a CloudFront distribution placed in front of the API.

**Why it matters:** For APIs returning large JSON payloads, gzip compression achieves 60–80% size reduction. Without it, data transfer costs and client parse time are higher than they need to be. The implementation point is simply different from REST API.

Handle compression in your Lambda handler for full control:

```csharp
using System.IO.Compression;

private static APIGatewayHttpApiV2ProxyResponse CompressedResponse<T>(T body, string acceptEncoding)
{
    var json = JsonSerializer.Serialize(body);

    if (acceptEncoding?.Contains("gzip") == true)
    {
        using var ms = new MemoryStream();
        using (var gz = new GZipStream(ms, CompressionLevel.Optimal))
        using (var sw = new StreamWriter(gz))
            sw.Write(json);

        return new APIGatewayHttpApiV2ProxyResponse
        {
            StatusCode = 200,
            Headers = new Dictionary<string, string>
            {
                { "Content-Type", "application/json" },
                { "Content-Encoding", "gzip" }
            },
            Body = Convert.ToBase64String(ms.ToArray()),
            IsBase64Encoded = true
        };
    }

    return new APIGatewayHttpApiV2ProxyResponse
    {
        StatusCode = 200,
        Headers = new Dictionary<string, string> { { "Content-Type", "application/json" } },
        Body = json
    };
}

// In your handler:
var acceptEncoding = request.Headers.TryGetValue("accept-encoding", out var ae) ? ae : null;
return CompressedResponse(result, acceptEncoding);
```

Alternatively, if you have CloudFront in front of your HTTP API, enable compression at the CloudFront level with a cache policy that includes `Accept-Encoding` in the cache key — CloudFront will then compress responses automatically with no Lambda code required.

* * *

### Set Maximum Payload Size Limits

**What it is:** HTTP API enforces a hard service limit of 10 MB per request payload. You should enforce a much lower limit appropriate for your API's actual use case, rejecting oversized requests in your Lambda handler before any processing begins.

**Why it matters:** Accepting 10 MB request bodies by default invites abuse — an attacker can send max-size payloads to exhaust Lambda memory or processing time on payloads your business logic will reject anyway. Check the `Content-Length` header or measure the body size at handler entry point.

* * *

### Export Your API as OpenAPI and Publish Documentation

**What it is:** HTTP API can export its definition as an OpenAPI 3.0 document via the AWS CLI or SDK. This document can feed Swagger UI, Postman, NSwag, Kiota, or any OpenAPI-compatible toolchain.

**Why it matters:** A machine-readable contract enables type-safe .NET client generation (via NSwag or Kiota), automated contract tests, and a documentation portal. It also serves as the authoritative source of truth for route definitions, authorizer requirements, and response schemas.

```bash
aws apigatewayv2 export-api \
  --api-id abc123def4 \
  --output-type OAS30 \
  --specification OAS30 \
  --stage-name '$default' \
  api-definition.yaml
```

## Quick Reference

> **Must** = required for any production HTTP API regardless of workload. **Optional** = strongly advisable but context-dependent.

| # | Practice | Priority | When to apply |
| --- | --- | --- | --- |
| **Security** |  |  |  |
| 1 | Use the native JWT authorizer | Must |  |
| 2 | Use Lambda authorizer for custom auth | Optional | When JWT validation alone is insufficient |
| 3 | Protect Your HTTP API with WAF via CloudFront | Optional | Any publicly accessible API |
| 4 | Configure CORS with explicit origins | Must | Any API called from browser clients |
| 5 | Enforce TLS 1.2 on custom domains | Must |  |
| **Throttling** |  |  |  |
| 6 | Set stage-level throttle limits | Must |  |
| **Request Validation** |  |  |  |
| 7 | Validate payloads in your .NET handler | Must | Always — HTTP API has no native body validator |
| **Integration** |  |  |  |
| 8 | Use Lambda proxy integration with v2.0 payload format | Must | All new Lambda-backed routes |
| 9 | Never return HTTP 200 for errors | Must | Always |
| 10 | Set explicit integration timeout | Must | Always — align to Lambda timeout + 1–2s margin |
| 11 | Use VPC Link v2 for private integrations | Must | When backend is inside a VPC |
| **Observability** |  |  |  |
| 12 | Enable access logging with JSON format | Must | Always |
| 14 | Set alarms on 5XX, latency, and count | Must | Always |
| 15 | Set log retention on all log groups | Must | Always |
| **Deployment & Stages** |  |  |  |
| 16 | Use stage variables | Must | Any API deployed to multiple stages |
| 17 | Use custom domain names | Must | Always — never expose the default execute-api URL |
| **Cost Optimization** |  |  |  |
| 18 | Use CloudFront in front for caching | Optional | Read-heavy public APIs with cacheable GET responses |
| 19 | Tag all resources with cost allocation tags | Must | Any shared or multi-team AWS account |
| **Operations** |  |  |  |
| 20 | Return consistent error responses from Lambda — no custom gateway responses | Must | Any API called from browser clients; accept that gateway-level rejections have a fixed body format |
| 21 | Handle compression in Lambda or CloudFront — no native HTTP API compression | Optional | APIs returning large JSON payloads (> 1 KB average) |
| 22 | Set maximum payload size limits | Optional |  |
| 23 | Export OpenAPI and publish documentation | Must | Any API consumed by other teams or SDK generation |

This checklist is a starting point. Adapt it to your team's compliance requirements and architectural patterns. The goal is a shared, team-maintained definition of what a production-ready HTTP API looks like for your organization. Thanks, and happy coding.