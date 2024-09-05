---
title: "Understanding Amazon API Gateway: Methods and Integrations"
datePublished: Sun Sep 01 2024 16:32:31 GMT+0000 (Coordinated Universal Time)
cuid: cm0jsibtu003w08l64v9xdpix
slug: understanding-amazon-api-gateway-methods-and-integrations
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1724718234865/cbe3e6a6-75fd-4b74-bedf-b7934c948742.png
tags: aws, aws-apigateway, amazon-api-gateway

---

> Amazon API Gateway is an AWS service for creating, publishing, maintaining, monitoring, and securing REST, HTTP, and WebSocket APIs at any scale.

Working with Amazon API Gateway can be overwhelming, especially at the beginning when many concepts are unclear. This lack of knowledge can cause us to miss many cool features, leading to underusing the power we have at our disposal. When a request reaches Amazon API Gateway, it goes through multiple stages from the client to the backend service and back to the client.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1724788364299/498f1b56-6e78-4809-98a5-734093c998e8.png align="center")

In the image above, we can see all the stages that will be reviewed in this post to gain an overview of the capabilities of Amazon API Gateway.

## Method Request

The [method request](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-method-settings-method-request.html) section defines how to handle incoming requests before they are sent to the backend. It includes authorization settings, API KEY checks, and request validation (performed in that order).

### Authorization settings

Amazon API Gateway offers several ways to secure our API:

* [IAM Roles](https://docs.aws.amazon.com/apigateway/latest/developerguide/permissions.html): Use AWS Identity and Access Management (IAM) roles to control who can access our API.
    
* [Amazon Cognito User Pools](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-integrate-with-cognito.html): Amazon Cognito manages user authentication and authorization.
    
* [Custom Authorizers](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-use-lambda-authorizer.html): Use a Lambda function to run custom logic to authenticate and authorize requests.
    

### API KEY Check

We can configure Amazon API Gateway to require an API Key as part of the request header. This API KEY identifies the request caller and applies a specific [usage plan](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-api-usage-plans.html) to them. A usage plan primarily manages two aspects: throttling limits and quota limits.

### Request Validation

Amazon API Gateway allows us to [validate](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-method-request-validation.html) parameters like query strings and headers as required. We can also validate the body of our request through a model. The model specifies the expected structure of the payload, ensuring it adheres to a specific schema.

## Integration Request

The [integration request](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-integration-settings-integration-request.html) section defines how to forward and transform incoming requests to the backend.

### Forward

The [integration type](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-api-integration-types.html) determines how method request data is forwarded to the backend and how the response data from the backend is returned to the method response. Integration types include:

* [AWS\_PROXY](https://docs.aws.amazon.com/apigateway/latest/developerguide/set-up-lambda-proxy-integrations.html#api-gateway-simple-proxy-for-lambda-output-format): Also known as Lambda proxy, this type does not require configuring the integration request or response. AWS API Gateway forwards the client's request to the Lambda function using a [predefined format](https://docs.aws.amazon.com/apigateway/latest/developerguide/set-up-lambda-proxy-integrations.html#api-gateway-simple-proxy-for-lambda-input-format). Similarly, to return the Lambda function's result to the client, the Lambda function must use a [predefined format](https://docs.aws.amazon.com/apigateway/latest/developerguide/set-up-lambda-proxy-integrations.html#api-gateway-simple-proxy-for-lambda-output-format).
    
* AWS: This type forwards a request to another AWS service. In this setup, we must configure the integration request and response. It is necessary to map data from the method request to the integration request and from the integration response to the method response.
    
* [HTTP\_PROXY](https://docs.aws.amazon.com/apigateway/latest/developerguide/setup-http-integrations.html#api-gateway-set-up-http-proxy-integration-on-proxy-resource): This type integrates with an HTTP endpoint without setting up the integration request or response. API Gateway forwards the incoming request from the client to the HTTP endpoint and sends the response from the HTTP endpoint back to the client.
    
* [HTTP](https://docs.aws.amazon.com/apigateway/latest/developerguide/setup-http-integrations.html#set-up-http-custom-integrations): Also known as HTTP custom integration. In this setup, we must configure the integration request and response.
    

### Transform

This feature allows us to [transform](https://docs.aws.amazon.com/apigateway/latest/developerguide/rest-api-data-transformations.html) the client request into the backend request. As mentioned earlier, this feature is only available for the AWS and HTTP integration types. We can [map](https://docs.aws.amazon.com/apigateway/latest/developerguide/request-response-data-mappings.html#mapping-request-parameters) method request data to the integration request parameters (path, query string, and headers). The [mapping](https://docs.aws.amazon.com/apigateway/latest/developerguide/models-mappings.html) between the method request payload and the integration request payload is done using the [Velocity Template Language (VTL)](https://velocity.apache.org/engine/devel/vtl-reference.html). Each mapping template is defined by the content type received in the request. If a request does not have a defined mapping template, the [passthrough behavior](https://docs.aws.amazon.com/apigateway/latest/developerguide/integration-passthrough-behaviors.html) option will describe how Amazon API Gateway handles the incoming request:

* WHEN\_NO\_MATCH: The request is passed through to the backend without transformation when the content type does not match any content types associated with the mapping templates.
    
* NEVER: The request is rejected if no template matches.
    
* WHEN\_NO\_TEMPLATES: The request is passed through to the backend without transformation only when no mapping template is defined in the integration request.
    

## Integration Response

The [integration response](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-integration-settings-integration-response.html) section defines how the response from the backend integration is processed before being returned to the client. It transforms the backend response into a format that our method response can handle.

### Transform

Like the integration request, we can [map](https://docs.aws.amazon.com/apigateway/latest/developerguide/request-response-data-mappings.html#mapping-response-parameters) integration response data (path, query string, and headers) to the method response headers and use mapping templates (VTL template) to map payloads. When configuring integration responses, we define a regex pattern to select a backend response and match it with a specific status code defined by the method response.

## Method Response

The [method response](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-method-settings-method-response.html) section defines the response that the client will receive. We can define multiple responses by status code, and for each one, we can specify headers and a model for the body. This information is useful for generating SDKs or documentation for our API.

> We must create method responses before the corresponding integration response because we use the method response status code there.

We hope this article is helpful, at least for a high-level understanding of what Amazon API Gateway can do. Stay tuned for more articles that will explore each section in detail, offering insights and practical examples. Thanks, and happy coding.